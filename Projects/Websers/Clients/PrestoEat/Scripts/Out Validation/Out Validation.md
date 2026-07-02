---
tags: [prestoeat, odoo, script, stock-picking, invoicing, automation]
---

# 📤 PrestoEat — Out Validation Script

This script automates the validation of outgoing delivery pickings in Odoo 19 and triggers customer invoice creation. It imports an Excel sheet of updated stock moves, validates the pickings, collects the linked Sale Orders, and creates and posts invoices dated to the original order date.

---

## 🚀 Execution Guide for the Technical Team

The script can be executed directly inside Odoo via a **Scheduled Action (Cron)** or a **Server Action**.

### Prerequisites

1. **Upload the Data Sheet**:
   Attach the target Excel (`.xlsx`) file named exactly **`out_automation.xlsx`** to the Scheduled Action or Server Action running this script.
   - Only `.xlsx` format is accepted. Any other format (`.xls`, `.csv`, etc.) will halt execution with an error.
   - The script searches for an attachment named `out_automation.xlsx` linked to a Scheduled Action (`ir.cron`) or Server Action (`ir.actions.server`).
   - If not found by exact name, it falls back to the most recently created attachment on any such action.
2. **Column Structure**:
   The sheet must contain at least the following columns:

   | Column | Purpose |
   |---|---|
   | **Stock Moves/ID** | Odoo External ID for matching the `stock.move` record |
   | **Stock Moves/Quantity** or **Quantity** | The quantity to set as completed/picked |
   | **Source Document** *(optional)* | Sale Order name (e.g. `SO4041`) used as a fallback to link SOs for invoicing |

---

### Step-by-Step Execution Workflow

#### Phase 1: Dry-Run Verification (Highly Recommended)
Always perform a dry run first to verify column mappings and confirm which records would be affected — **nothing is written to the database**.

1. Copy the script below into the Odoo Action Python code editor.
2. Set the toggle at the top of the script:
   ```python
   DRY_RUN = True
   ```
3. Run the script.
4. Open the Odoo server logs and confirm:
   - The correct file was detected (e.g., `Processing file: 'out_automation.xlsx'`).
   - Columns were mapped correctly (e.g., `Mapping Column Indices - ID: 1, Qty: 4, Source Doc: 5`).
   - The expected number of records were matched (e.g., `Import matched 20 stock.move record(s)`).
   - No errors were reported.

#### Phase 2: Live Execution
Once the dry-run output is validated, execute the live run:

1. Toggle the dry-run setting to `False`:
   ```python
   DRY_RUN = False
   ```
2. Run the script.
3. Check Odoo server logs to verify:
   - Picking validation: `Validated Picking 'WH/OUT/XXXXX' (X line(s) processed)`.
   - Invoice creation: `Created and Posted Invoice '[INV/XXXXX]' for Order 'SOXXXXX' dated YYYY-MM-DD`.
   - On error: `Failed to validate picking 'WH/OUT/XXXXX'. Error: ...` or `Failed to invoice/post Order 'SOXXXXX'. Error: ...`.

> [!NOTE]
> If an active invoice already exists for a Sale Order, the script skips it and logs: `Order 'SOXXXXX' already has active invoice(s) [INV/XXXXX] — skipping.`

---

## 📜 Script Code

```python
# ==========================================================
# Toggle: run once with DRY_RUN = True to sanity-check the
# column mapping/matching before it touches real records.
# ==========================================================
DRY_RUN = False

# ==========================================================
# 1. Locate the Excel attachment on this Scheduled Action.
#    Only .xlsx files are accepted.
# ==========================================================
attachment = env['ir.attachment'].sudo().search([
    ('res_model', 'in', ['ir.cron', 'ir.actions.server']),
    ('name', '=', 'out_automation.xlsx')
], limit=1, order='create_date desc')

if not attachment:
    attachment = env['ir.attachment'].sudo().search([
        ('res_model', 'in', ['ir.cron', 'ir.actions.server'])
    ], limit=1, order='create_date desc')

if not attachment:
    raise UserError(
        "Execution halted: No file found attached to any Scheduled Action or Server Action."
    )

_logger.info(f"Processing file: '{attachment.name}' (ID: {attachment.id})")

raw_data = attachment.raw or b''
if not raw_data:
    raise UserError("Execution halted: The attached file is completely empty (0 bytes).")

is_xlsx = raw_data.startswith(b'PK\x03\x04')

if is_xlsx:
    file_type = 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
else:
    raise UserError("Execution halted: The attached file is not an Excel file (.xlsx).")

# ==========================================================
# 2. Run the update through Odoo's OWN import engine.
# ==========================================================
import_wizard = env['base_import.import'].sudo().with_context({}).create({
    'res_model': 'stock.move',
    'file': raw_data,
    'file_name': attachment.name,
    'file_type': file_type,
})

options = {
    'has_headers': True,
    'quoting': '"',
    'encoding': 'utf-8',
    'separator': '',
}

preview = import_wizard.parse_preview(options)
headers = preview.get('headers') or []

template_headers = [
    'Destination Location',
    'Stock Moves/ID',
    'Stock Moves/Product/Internal Reference',
    'Stock Moves/Demand',
    'Stock Moves/Quantity',
    'Presto Order Reference',
]

if not isinstance(headers, list) or not headers:
    _logger.warning("parse_preview() returned no usable headers — falling back to the known template column order.")
    headers = template_headers

# Dynamically locate the columns we actually need
id_idx = -1
qty_idx = -1
src_doc_idx = -1

for idx, h in enumerate(headers):
    h_clean = str(h).strip().lower()
    if h_clean == 'stock moves/id' or (id_idx == -1 and 'stock moves/id' in h_clean):
        id_idx = idx
    elif h_clean in ('stock moves/quantity', 'quantity') or (qty_idx == -1 and 'quantity' in h_clean):
        qty_idx = idx
    elif h_clean == 'source document' or (src_doc_idx == -1 and 'source document' in h_clean):
        src_doc_idx = idx

if id_idx == -1:
    id_idx = 1  # Fallback position from template
if qty_idx == -1:
    qty_idx = 4  # Fallback position from template

_logger.info(f"Headers: {headers}")
_logger.info(f"Mapping Column Indices - ID: {id_idx}, Qty: {qty_idx}, Source Doc: {src_doc_idx}")

# Map fields for Odoo's native importer.
# 'id' matches by External ID. All other columns are skipped (False).
field_mapping = [False] * len(headers)
field_mapping[id_idx] = 'id'
field_mapping[qty_idx] = 'quantity'

result = import_wizard.execute_import(field_mapping, headers, options, dryrun=DRY_RUN)

errors = [m for m in result.get('messages', []) if m.get('type') == 'error']
if errors:
    _logger.error(f"Import reported {len(errors)} error(s): {errors[:10]}")

if not result.get('ids'):
    raise UserError(f"Execution halted: 0 stock.move records were matched.\nMessages: {result.get('messages')}")

moves = env['stock.move'].sudo().browse(result['ids']).exists()
_logger.info(f"Import matched {len(moves)} stock.move record(s).")

if DRY_RUN:
    _logger.info("🧪 DRY_RUN is True — nothing was written, stopping before picking validation.")
else:
    # ==========================================================
    # 3. Group updated moves by picking and validate each one (Out delivery)
    # ==========================================================
    picking_moves_map = {}
    for move in moves:
        if not move.picking_id:
            _logger.warning(f"Stock move ID {move.id} has no picking — skipped.")
            continue
        picking_moves_map.setdefault(move.picking_id, env['stock.move'])
        picking_moves_map[move.picking_id] |= move

    _logger.info(f"🔍 {len(moves)} move(s) span {len(picking_moves_map)} picking(s).")

    orders_to_invoice = env['sale.order']

    for picking, target_moves in picking_moves_map.items():
        try:
            if picking.state in ('draft', 'waiting', 'confirmed'):
                picking.action_assign()

            # Assign quantities to moves inside the picking
            for move in picking.move_ids:
                if move in target_moves:
                    qty = move.quantity if move.quantity > 0 else move.product_uom_qty
                    move.write({'quantity': qty, 'picked': True})
                else:
                    move.write({'quantity': 0, 'picked': False})

            res = picking.button_validate()

            # Handle immediate transfer or backorder wizard bypasses
            if isinstance(res, dict) and res.get('res_model'):
                wizard_ctx = dict(res.get('context', {}))
                wizard_ctx.update({
                    'active_model': 'stock.picking',
                    'active_id': picking.id,
                    'active_ids': [picking.id],
                })
                wizard = env[res['res_model']].sudo().with_context(wizard_ctx).create({})
                if hasattr(wizard, 'process'):
                    wizard.process()
                elif hasattr(wizard, 'action_confirm'):
                    wizard.action_confirm()

            _logger.info(f"Validated Picking '{picking.name}' ({len(target_moves)} line(s) processed).")

            # Collect unique Sale Orders linked to the target moves
            for move in target_moves:
                if move.sale_line_id and move.sale_line_id.order_id:
                    orders_to_invoice |= move.sale_line_id.order_id

        except Exception as e:
            _logger.error(f"Failed to validate picking '{picking.name}'. Error: {str(e)}")

    # ==========================================================
    # 4. Fallback: Parse sheet data directly to catch any missing Sale Orders by Name
    # ==========================================================
    if src_doc_idx != -1:
        raw_rows = preview.get('preview') or []
        for row in raw_rows:
            if len(row) > src_doc_idx and row[src_doc_idx]:
                so_name = str(row[src_doc_idx]).strip()
                if so_name and so_name.startswith(('SO', 'S0')):  # Matches S04041 or SO4041
                    matching_so = env['sale.order'].sudo().search([('name', '=', so_name)], limit=1)
                    if matching_so:
                        orders_to_invoice |= matching_so

    # ==========================================================
    # 5. Create, Date, and Post Invoices for the resolved Sale Orders
    # ==========================================================
    _logger.info(f"Found {len(orders_to_invoice)} unique Sale Order(s) to invoice.")

    for order in orders_to_invoice:
        # Avoid double-invoicing if an active invoice already exists
        existing_invoices = order.invoice_ids.filtered(lambda i: i.state != 'cancel')
        if existing_invoices:
            _logger.info(f"Order '{order.name}' already has active invoice(s) {existing_invoices.mapped('name')} — skipping.")
            continue

        try:
            # Generate the customer invoice natively
            invoices = order._create_invoices()

            if invoices:
                # Force invoice date to match the Sale Order's creation date
                invoice_date = order.date_order.date()
                invoices.write({'invoice_date': invoice_date})

                # Post the invoice natively to validate the accounting entries
                invoices.action_post()
                _logger.info(f"Created and Posted Invoice '{invoices.mapped('name')}' for Order '{order.name}' dated {invoice_date}.")
            else:
                _logger.warning(f"No invoice generated for Order '{order.name}'. Verify its invoicing policy settings.")

        except Exception as e:
            _logger.error(f"Failed to invoice/post Order '{order.name}'. Error: {str(e)}")
```
