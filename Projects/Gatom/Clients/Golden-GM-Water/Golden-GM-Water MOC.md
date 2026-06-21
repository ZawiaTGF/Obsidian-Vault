---
tags: [moc, client, erpnext, frappe, gatom, golden-gm-water]
---

# 🗺 Golden-GM-Water — Map of Content

> **Client**: Golden-GM-Water
> **Agency**: [[../../Gatom MOC|Gatom]]
> **Project**: ERPNext Implementation (Standard modules + HR Customizations)
> **Stack**: ERPNext v16 (Frappe), standard modules, custom HR configuration
> **Status**: 🟢 Active

---

## 🏞 Project Overview

Golden-GM-Water is a drinking water distribution business. The business model is straightforward:
- **Buy** drinking water (including the bottles) from a single wholesale vendor.
- **Stock** and manage the bottle inventory.
- **Sell** and distribute to multiple retail/corporate vendors.
- **Shipping**: Handled in-house (shipping applies, delivered by the business's own vehicle/drivers).
- **Team**: 2-3 employees manage all operations (sales, delivery, inventory, and accounts).

---

## 🧩 Business Domains

### 📦 Inventory & Stock
- Track drinking water bottles as inventory items.
- Monitor stock levels in the main warehouse.
- Handle returns of empty bottles (if applicable as deposit items).

### 🛒 Purchasing
- Manage relationship with the single wholesale vendor.
- Purchase Orders (POs), Purchase Receipts, and Purchase Invoices.

### 💼 Sales & Shipping
- Handle sales orders and customer invoices.
- **Delivery Trips**: Plan deliveries, assign drivers, and record shipping charges.
- Manage customer accounts (outstanding balances, credit limits).

### 👥 Human Resources (Customized)
- Employee master records for 2-3 staff members.
- Driver-specific fields (License No, expiry).
- Delivery commission tracking / payroll.
- Basic attendance and leave logging.

---

## 📄 Documents

| Document | Purpose |
|---|---|
| [[Foundation]] | Original project brief and constraints |
| [[Golden-GM-Water ERPNext Setup and Workflow]] | Complete functional configuration guide mapping standard modules and specifying HR customizations |
| [[Golden-GM-Water ERPNext Setup and Workflow (Arabic)]] | Arabic version of the functional configuration guide |


---

## 🔗 Related

- [[../../Gatom MOC|↑ Gatom]] — the agency managing this project
- [[../Golden-GM/Golden-GM MOC|Golden-GM]] — sister/sibling project
- [[../../../../../Home|🏠 Home]]
