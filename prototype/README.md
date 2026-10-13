# Service UI prototype — the design reference

`index.html` is the agreed design for the service roles (waiter, cashier, cook, inventory lead, F&B Store,
F&B Manager). The frontend built from [SERVICE_FRONTEND_PLAN.md](../docs/SERVICE_FRONTEND_PLAN.md) must look and
read **exactly** like it. It is also published at https://claude.ai/artifact/WeHAmEamDt29kMrLPVcCrV; this file is
the copy that counts.

## Rules

- **Frozen while the frontend is built.** Don't edit `index.html` to match what the app does. A design change is
  made here first, agreed, and then built; record it at the bottom of this file.
- **What it decides:** layout, spacing, sizes, type, colour, icons, motion, wording (headings, buttons, toasts,
  empty states, errors), and the responsive breakpoints.
- **What it doesn't:** data and behaviour come from [SERVICE_API_CONTRACT.md](../docs/SERVICE_API_CONTRACT.md).
  The demo data, the simulation and the "Prototype" tab are not part of the product.

## Running it

Start the `ims-ui-prototype` entry in `.claude/launch.json` (a static server on http://localhost:8765). Opening the
file directly works too, but browsers block storage for `file:`/`data:` pages, so sessions and cross-tab sync won't.

For side-by-side checks:
1. Open the **Prototype** tab on the right edge → turn **Simulate the rest of the team** off, then **Reset demo
   data**, so nothing moves between screenshots.
2. Use **Appearance** in the same tab for Light / Dark.
3. Jump to any person from the list in that tab (skips sign-in). To see sign-in itself, use **Back to the sign-in
   start**.

Demo sign-in: any four digits as a PIN; office accounts `ali.raza@rykards.pk` (F&B Manager) and
`usman.ghani@rykards.pk` (F&B Store) with any password. Test values only.

## Where each screen lives in the code

| Screen | Component(s) in `index.html` |
|---|---|
| Start, device set-up, office sign-in | `Entry`, `AccountForm` |
| PIN screen, PIN pad | `SignIn`, `PinPad` |
| Waiter | `WaiterApp`, `Floor`, `SeatSheet`, `OrderScreen`, `LineRow`, `ItemSheet`, `LineSheet`, `OpenOrders`, `Pickups` |
| Kitchen | `KitchenApp`, `Ticket`, `AllDay`, `PassView` |
| Cashier | `CashierApp`, `BillCard`, `Settle`, `DiscountSheet`, `Receipt`, `CashPayments`, `CloseRegister` |
| Inventory lead | `InventoryApp`, `InvOverview`, `InvStock`, `ItemDrawer`, `Composer`, `InvRequests`, `ReqDrawer`, `InvCount`, `InvWaste` |
| F&B Store | `StoreApp`, `StoreOverview`, `StoreDispatch`, `PickDrawer`, `StoreReceiving`, `ReceiveDrawer`, `StorePurchases`, `PrComposer`, `StoreStock` |
| F&B Manager | `ManagerApp`, `MgrOverview`, `MgrApprovals`, `ApprovalSheet`, `MgrSales`, `MgrSalesToday`, `SalesExtra`, `MgrPurchasing`, `PoDrawer`, `BillDrawer`, `PurchExtra`, `NewPoSheet`, `MgrStock`, `LevelsDrawer`, `MgrCosting`, `MgrReports`, `MgrTeam`, `DevicesPanel`, `AddPerson`, `ShiftSheet`, `MgrSetup` |
| Shared | `DeskShell`, `Sheet`, `Drawer`, `Stepper`, `UTabs`, `Tog`, `RTable`, `BarChart`, `HB`, `HourChart`, `Level`, `Move`, `Toasts`, `Icon` (paths in `P`) |

All styling is in the `<style>` block at the top; the tokens are the `:root` blocks.

## Design changes
- (none yet)
