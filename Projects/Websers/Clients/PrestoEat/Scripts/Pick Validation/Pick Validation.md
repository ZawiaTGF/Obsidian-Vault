---
tags: [prestoeat, odoo, script, stock-picking, automation]
---

# 📦 PrestoEat — Pick Validation Script

This script automates the validation of stock pickings in Odoo 19 by importing an Excel/CSV sheet containing updated stock moves, mapping the items by their External ID, writing the validated quantities, and automatically validating the pickings.

---

## 🚀 Execution Guide for the Technical Team

The script can be executed directly inside Odoo via a **Scheduled Action (Cron)**, a **Server Action**, or the **Odoo Shell**.

### Prerequisites

1. **Upload the Data Sheet**: 
   Attach the target Excel (`.xlsx`) file to the Scheduled Action or Server Action running this script.
   - The script searches for attachments linked to Scheduled Actions (`ir.cron`) or Server Actions (`ir.actions.server`).
   - By default, it looks for files named `pick_automation.xlsx`; if not found, it falls back to the most recently created attachment on any action.
2. **Column Structure**: 
   The sheet must contain at least the following columns:
   - **Stock Moves/ID** (Odoo External ID for matching the `stock.move` record)
   - **Stock Moves/Quantity** or **Quantity** (The quantity to set as completed/picked)

---

### Step-by-Step Execution Workflow

#### Phase 1: Dry-Run Verification (Highly Recommended)
Always perform a dry run first to verify column mappings and see which records would be affected.

1. Copy the script below into the Odoo Action Python code editor or shell.
2. Set the toggle at the top of the script:
   ```python
   DRY_RUN = True
   ```
3. Run the script.
4. Open the Odoo server logs (or check the output) and confirm:
   - The correct file was detected (e.g., `Processing file: 'pick_automation.xlsx'`).
   - The headers were matched correctly (e.g., `Mapping column 1 ('Stock Moves/ID') -> id, column 4 ('Stock Moves/Quantity') -> quantity`).
   - The correct number of records were matched (e.g., `Import matched 15 stock.move record(s)`).
   - Ensure no errors were reported.

#### Phase 2: Live Execution
Once the dry-run output is validated, execute the live run:

1. Toggle the dry-run setting to `False`:
   ```python
   DRY_RUN = False
   ```
2. Run the script.
3. Check Odoo server logs to verify picking validation:
   - Verify log entries matching: `Validated picking 'WH/OUT/XXXXX' (X line(s) from the sheet)`.
   - On error, validation failure messages will be logged under `Failed to validate picking 'WH/OUT/XXXXX'`.

---

## 📜 Script Code

```python
# ==========================================================
# Toggle: run once with DRY_RUN = True to sanity-check the
# column mapping/matching before it touches real records.
# ==========================================================
DRY_RUN = False

# ==========================================================
# 1. Locate the Excel/CSV attachment on this Scheduled Action
# ==========================================================
attachment = env['ir.attachment'].sudo().search([
    ('res_model', 'in', ['ir.cron', 'ir.actions.server']),
    ('name', '=', 'pick_automation.xlsx')
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
    raise UserError("Execution halted: The attached file is not an Excel file (xlsx).")

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
    'separator': ',' if file_type == 'text/csv' else '',
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

# Dynamically locate the two columns we actually need
id_idx = -1
qty_idx = -1
for idx, h in enumerate(headers):
    h_clean = str(h).strip().lower()
    if h_clean == 'stock moves/id' or (id_idx == -1 and 'stock moves/id' in h_clean):
        id_idx = idx
    elif h_clean in ('stock moves/quantity', 'quantity') or (qty_idx == -1 and 'quantity' in h_clean):
        qty_idx = idx

if id_idx == -1:
    id_idx = 1  # fixed fallback position from your template
if qty_idx == -1:
    qty_idx = 4  # fixed fallback position from your template

_logger.info(f"Headers: {headers}")
_logger.info(f"Mapping column {id_idx} ('{headers[id_idx]}') -> id, column {qty_idx} ('{headers[qty_idx]}') -> quantity")

# 'id' is Odoo's special "match by External ID" field.
# All other columns are skipped (False) so nothing else gets overwritten.
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
    _logger.info("DRY_RUN is True — nothing was written, stopping before picking validation.")
else:
    # ==========================================================
    # 3. Group updated moves by picking and validate each one
    #    the same way the "Validate" button does.
    # ==========================================================
    picking_moves_map = {}
    for move in moves:
        if not move.picking_id:
            _logger.warning(f"Stock move ID {move.id} has no picking — skipped.")
            continue
        picking_moves_map.setdefault(move.picking_id, env['stock.move'])
        picking_moves_map[move.picking_id] |= move

    _logger.info(f"{len(moves)} move(s) span {len(picking_moves_map)} picking(s).")


    for picking, target_moves in picking_moves_map.items():
        try:
            if picking.state in ('draft', 'waiting', 'confirmed'):
                picking.action_assign()

            # Using Odoo 19 core move_ids list field natively
            for move in picking.move_ids:
                if move in target_moves:
                    qty = move.quantity if move.quantity > 0 else move.product_uom_qty
                    move.write({'quantity': qty, 'picked': True})
                else:
                    move.write({'quantity': 0, 'picked': False})

            res = picking.button_validate()

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

            _logger.info(f"Validated picking '{picking.name}' ({len(target_moves)} line(s) from the sheet).")

        except Exception as e:
            _logger.error(f"Failed to validate picking '{picking.name}'. Error: {str(e)}")
```
