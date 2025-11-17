# Queries and Functions Documentation

This document lists key queries, functions, and proposed edits that implement sales-per-product reporting, dashboard revenue, and loyalty metrics, including biller-based scoping and per-product filtering. Exact file:line references and source paths are provided.

## Conventions
- References use `file_path:line_number` for traceability.
- Multi-tenant filters use `pos_accnt_id`; biller scoping adds `biller_id`.
- Warehouse and product filters apply when provided.

## Routes
- Dashboard routes: `routes/web.php:170–185`
- Report routes: `routes/web.php:497–561`

## Laravel Controllers

- dailySale
  - Source: `app/Http/Controllers/ReportController.php:246–295`
  - Query: `Sale::whereDate('created_at', $date)->where('pos_accnt_id', $pos_accnt_id)->selectRaw(...)` at `app/Http/Controllers/ReportController.php:270–273`
  - Comment: Aggregates daily totals; scoped by `pos_accnt_id`. Proposed fix adds `where('biller_id', $biller_id)`.

- dailySaleByWarehouse
  - Source: `app/Http/Controllers/ReportController.php:297–355`
  - Query: `Sale::where('warehouse_id', $data['warehouse_id'])->where('pos_accnt_id', $pos_accnt_id)->whereDate('created_at', $date)->selectRaw(...)` at `app/Http/Controllers/ReportController.php:332–335`
  - Comment: Same aggregation filtered by warehouse; proposed fix adds `biller_id` constraint.

- profitLoss
  - Source: `app/Http/Controllers/ReportController.php:837–969`
  - Queries:
    - Product sales join: `Product_Sale::join('sales', ...)` with `sales.pos_accnt_id` at `app/Http/Controllers/ReportController.php:859–866`
    - Purchase sum: `Purchase::whereDate(...)->where('pos_accnt_id', $pos_accnt_id)->selectRaw(...)` at `app/Http/Controllers/ReportController.php:876–879`
    - Sale sum: `Sale::whereDate(...)->where('pos_accnt_id', $pos_accnt_id)->selectRaw(...)` at `app/Http/Controllers/ReportController.php:886–889`
    - Return sum: `Returns::whereDate(...)->where('pos_accnt_id', $pos_accnt_id)->selectRaw(...)` at `app/Http/Controllers/ReportController.php:896–900`
    - ReturnPurchase sum: `ReturnPurchase::whereDate(...)->where('pos_accnt_id', $pos_accnt_id)->selectRaw(...)` at `app/Http/Controllers/ReportController.php:906–910`
    - Expense sum: `Expense::whereDate(...)->where('pos_accnt_id', $pos_accnt_id)->sum('amount')` at `app/Http/Controllers/ReportController.php:916–919`
    - Income sum: `Income::whereDate(...)->where('pos_accnt_id', $pos_accnt_id)->sum('amount')` at `app/Http/Controllers/ReportController.php:921–924`
    - Payments join: `DB::table('payments')->join('sales', ...)` with `sales.pos_accnt_id` at `app/Http/Controllers/ReportController.php:955–969`
  - Comment: Computes KPIs and counts; proposed fix adds `biller_id` to each aggregate.

- saleReportData
  - Source: `app/Http/Controllers/ReportController.php:2358–2598`
  - Queries (warehouse_id == 0):
    - Sold amount per variant: `Product_Sale::where([...])->whereDate(...)->sum('total')` at `app/Http/Controllers/ReportController.php:2419–2423`
    - Sold qty per variant: `Product_Sale::select('sale_unit_id','qty')->where([...])->whereDate(...)->get()` at `app/Http/Controllers/ReportController.php:2424–2428`
    - Non-variant sold amount and qty at `app/Http/Controllers/ReportController.php:2456–2460` and `2461–2477`
  - Queries (warehouse branch):
    - Sold amount via join: `DB::table('sales')->join('product_sales', ...)` at `app/Http/Controllers/ReportController.php:2545–2550`
    - Sold qty via join: `select('product_sales.sale_unit_id','product_sales.qty')` at `app/Http/Controllers/ReportController.php:2551–2557`
  - Comment: Builds per-product rows; proposed fix adds biller and product filters.

- saleReportChart
  - Source: `app/Http/Controllers/ReportController.php:2616–2683`
  - Query: `DB::table('sales')->join('product_sales', ...)->where('sales.pos_accnt_id', $pos_accnt_id)->sum('product_sales.qty')` at `app/Http/Controllers/ReportController.php:2641–2646, 2658–2661`
  - Comment: Rolling sums by date points; supports warehouse and product list; proposed fix adds `biller_id` and single product filter.

- paymentReportByDate
  - Source: `app/Http/Controllers/ReportController.php:2685–2701`
  - Query: `Payment::whereDate(...)->where('pos_accnt_id', $pos_accnt_id)->get()` at `app/Http/Controllers/ReportController.php:2695–2698`
  - Comment: Lists payments by date; proposed fix adds `biller_id`.

- dashboardFilter
  - Source: `app/Http/Controllers/HomeController.php:548–687`
  - Queries:
    - Product sales aggregation with `sales.pos_accnt_id` at `app/Http/Controllers/HomeController.php:556–563`
    - Revenue: `Sale::whereDate(...)->where('pos_accnt_id', $pos_accnt_id)->sum(DB::raw('grand_total - shipping_cost'))` at `app/Http/Controllers/HomeController.php:568–573, 632–637, 649–654`
    - Returns: `Returns::whereDate(...)->where('pos_accnt_id', $pos_accnt_id)->sum('grand_total')` at `app/Http/Controllers/HomeController.php:575–579, 638–642, 655–659`
    - PurchaseReturn, Expense, Income sums in adjacent lines
  - Comment: Dashboard KPIs; proposed fix adds `biller_id` and optional `product_id`.

## Server Router (tRPC/Node)

- dashboardRouter.stats
  - Source: `uhaiui/src/server/routers/dashboard.ts:10–71`
  - Queries:
    - Revenue SQL with `pos_accnt_id` at `uhaiui/src/server/routers/dashb
oard.ts:25–31, 33–43`
    - Sale Return SQL: `returns` at `uhaiui/src/server/routers/dashboard.ts:49–68`
    - Purchase Return SQL: `return_purchases` at `uhaiui/src/server/routers/dashboard.ts:74–93`
    - Loyalty SQL: join `sales`→`product_sales`→`products` at `uhaiui/src/server/routers/dashboard.ts:116–146`
  - Comment: Proposed fix adds `biller_id` to SQL and optional `product_id` filter.

- dashboardRouter.warehouses
  - Source: `uhaiui/src/server/routers/dashboard.ts:159–186`
  - Query: `SELECT id,name FROM warehouses WHERE is_active = 1 AND pos_accnt_id = ?` at `uhaiui/src/server/routers/dashboard.ts:170–175`
  - Comment: Proposed fix adds `AND biller_id = ?`.

## Frontend (Next.js)

- Dashboard page: tRPC usage
  - Source: `uhaiui/src/app/dashboard/page.tsx:90–116`
  - Usage: `trpc.dashboard.stats.useQuery({ pos_accnt_id, start_date, end_date, warehouse_id? })` at `uhaiui/src/app/dashboard/page.tsx:100–106`
  - Comment: Proposed change passes `biller_id` and optional `product_id`.

- Dashboard cards display
  - Source: `uhaiui/src/app/dashboard/page.tsx:400–456`
  - Data: `stats?.revenue`, `stats?.saleReturn`, `stats?.purchaseReturn`, `stats?.loyaltyPoints`

## Blade Views

- Product report AJAX
  - Source: `resources/views/backend/report/product_report.blade.php:121–139`
  - Query: POST `product_report_data` with `start_date`, `end_date`, `warehouse_id` at `resources/views/backend/report/product_report.blade.php:125–133`
  - Comment: Proposed change adds `product_id` parameter.

- Sale report AJAX
  - Source: `resources/views/backend/report/sale_report.blade.php:101–116`
  - Query: POST `sale_report_data` with `start_date`, `end_date`, `warehouse_id` at `resources/views/backend/report/sale_report.blade.php:105–113`

## New Files and Edits

- Redwoods.md
  - Source: `Redwoods.md`
  - Comment: Full report with exact code lines, proposed fixes, and annotations added. Sections include routes, controllers, tRPC router, frontend, Blade, and authorization. Annotated changes explain added `biller_id` constraints, product filters, and admin visibility.

## Proposed Fixes Summary (with Comments)

- Biller scoping in controllers
  - Add: `where('biller_id', $biller_id)` alongside existing `pos_accnt_id` in `Sale`, `Returns`, `ReturnPurchase`, `Expense`, `Income`, and joined queries.
  - Files: `app/Http/Controllers/HomeController.php` (`dashboardFilter`), `app/Http/Controllers/ReportController.php` (`dailySale`, `dailySaleByWarehouse`, `profitLoss`, `productReportData`, `saleReportData`, `saleReportChart`, `paymentReportByDate`).
  - Comment: Ensures all KPIs and reports are tied to specific biller.

- Product filter support
  - Add: `product_id` to `productReportData`, `saleReportData`, `saleReportChart`, and tRPC `stats` input.
  - Files: `app/Http/Controllers/ReportController.php`, `uhaiui/src/server/routers/dashboard.ts`, `resources/views/backend/report/product_report.blade.php`, `uhaiui/src/app/dashboard/page.tsx`.
  - Comment: Enables filtering individual product data end-to-end.

- tRPC SQL updates
  - Add: `AND biller_id = ?` to `sales`, `returns`, `return_purchases` queries; extend loyalty join filter and optional `AND ps.product_id = ?`.
  - File: `uhaiui/src/server/routers/dashboard.ts`.
  - Comment: Server-side KPI consistency with biller scoping.

- Authorization
  - Gate: `view-all-data` in `AuthServiceProvider` to allow admin-only cross-biller visibility.
  - Controller guard: default `biller_id` to authenticated user unless admin overrides.
  - tRPC guard: derive `billerId` from context for non-admins.
  - Comment: Restricts non-admins to their own biller data.

## Notes
- All references exclude `productfix` paths.
- This documentation aligns with Redwoods.md and expands with comments for each query/function and proposed edits.
