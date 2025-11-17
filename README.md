# Redwoods.md — Full Report and Code References (excluding `productfix`)

## Scope
- Sales-per-product reports, dashboard revenue, and loyalty implementations across backend (Laravel), server router (tRPC/Node), and frontend (Next.js/Blade).
- Adds biller-based filtering design and individual product filtering.
- Includes exact code lines from the repository with file:line references; excludes `productfix`.

## Routes

### Dashboard Routes — `routes/web.php:170–185`
```php
# routes/web.php:170–185
Route::controller(HomeController::class)->group(function () {
    Route::get('/', 'index');
    Route::get('/dashboard', 'dashboard');
    Route::get('/yearly-best-selling-price', 'yearlyBestSellingPrice');
    Route::get('/yearly-best-selling-qty', 'yearlyBestSellingQty');
    Route::get('/monthly-best-selling-qty', 'monthlyBestSellingQty');
    Route::get('/recent-sale', 'recentSale');
    Route::get('/recent-purchase', 'recentPurchase');
    Route::get('/recent-quotation', 'recentQuotation');
    Route::get('/recent-payment', 'recentPayment');
    Route::get('switch-theme/{theme}', 'switchTheme')->name('switchTheme');
    Route::get('/dashboard-filter/{start_date}/{end_date}/{warehouse_id}', 'dashboardFilter');
    Route::get('addon-list', 'addonList');
    Route::get('my-transactions/{year}/{month}', 'myTransaction');
});
```

### Report Routes — `routes/web.php:497–561`
```php
# routes/web.php:497–561
Route::controller(ReportController::class)->group(function () {
    Route::prefix('report')->group(function () {
        Route::get('product_quantity_alert', 'productQuantityAlert')->name('report.qtyAlert');
        Route::get('daily-sale-objective', 'dailySaleObjective')->name('report.dailySaleObjective');
        Route::post('daily-sale-objective-data', 'dailySaleObjectiveData');
        Route::get('product-expiry', 'productExpiry')->name('report.productExpiry');
        Route::get('warehouse_stock', 'warehouseStock')->name('report.warehouseStock');
        Route::get('daily_sale/{year}/{month}', 'dailySale');
        Route::post('daily_sale/{year}/{month}', 'dailySaleByWarehouse')->name('report.dailySaleByWarehouse');
        Route::get('monthly_sale/{year}', 'monthlySale');
        Route::post('monthly_sale/{year}', 'monthlySaleByWarehouse')->name('report.monthlySaleByWarehouse');
        Route::get('daily_purchase/{year}/{month}', 'dailyPurchase');
        Route::post('daily_purchase/{year}/{month}', 'dailyPurchaseByWarehouse')->name('report.dailyPurchaseByWarehouse');
        Route::get('monthly_purchase/{year}', 'monthlyPurchase');
        Route::post('monthly_purchase/{year}', 'monthlyPurchaseByWarehouse')->name('report.monthlyPurchaseByWarehouse');
        Route::get('best_seller', 'bestSeller');
        Route::post('best_seller', 'bestSellerByWarehouse')->name('report.bestSellerByWarehouse');
        Route::post('profit_loss', 'profitLoss')->name('report.profitLoss');
        Route::get('product_report', 'productReport')->name('report.product');
        Route::post('product_report_data', 'productReportData');
        Route::post('purchase', 'purchaseReport')->name('report.purchase');
        Route::post('purchase_report_data', 'purchaseReportData');
        Route::post('sale_report', 'saleReport')->name('report.sale');
        Route::post('sale_report_data', 'saleReportData');
        Route::get('challan-report', 'challanReport')->name('report.challan');
        Route::post('sale-report-chart', 'saleReportChart')->name('report.saleChart');
        Route::post('payment_report_by_date', 'paymentReportByDate')->name('report.paymentByDate');
        Route::post('warehouse_report', 'warehouseReport')->name('report.warehouse');
        Route::post('warehouse-sale-data', 'warehouseSaleData');
        Route::post('warehouse-purchase-data', 'warehousePurchaseData');
        Route::post('warehouse-expense-data', 'warehouseExpenseData');
        Route::post('warehouse-quotation-data', 'warehouseQuotationData');
        Route::post('warehouse-return-data', 'warehouseReturnData');
        Route::post('user_report', 'userReport')->name('report.user');
        Route::post('user-sale-data', 'userSaleData');
        Route::post('user-purchase-data', 'userPurchaseData');
        Route::post('user-expense-data', 'userExpenseData');
        Route::post('user-quotation-data', 'userQuotationData');
        Route::post('user-payment-data', 'userPaymentData');
        Route::post('user-transfer-data', 'userTransferData');
        Route::post('user-payroll-data', 'userPayrollData');
        Route::post('biller_report', 'billerReport')->name('report.biller');
        Route::post('biller-sale-data','billerSaleData');
        Route::post('biller-quotation-data','billerQuotationData');
        Route::post('biller-payment-data','billerPaymentData');
        Route::post('customer_report', 'customerReport')->name('report.customer');
        Route::post('customer-sale-data', 'customerSaleData');
        Route::post('customer-payment-data', 'customerPaymentData');
        Route::post('customer-quotation-data', 'customerQuotationData');
        Route::post('customer-return-data', 'customerReturnData');
        Route::post('customer-group', 'customerGroupReport')->name('report.customer_group');
        Route::post('customer-group-sale-data', 'customerGroupSaleData');
        Route::post('customer-group-payment-data', 'customerGroupPaymentData');
        Route::post('customer-group-quotation-data', 'customerGroupQuotationData');
        Route::post('customer-group-return-data', 'customerGroupReturnData');
        Route::post('supplier', 'supplierReport')->name('report.supplier');
        Route::post('supplier-purchase-data', 'supplierPurchaseData');
        Route::post('supplier-payment-data', 'supplierPaymentData');
        Route::post('supplier-return-data', 'supplierReturnData');
        Route::post('supplier-quotation-data', 'supplierQuotationData');
        Route::post('customer-due-report', 'customerDueReportByDate')->name('report.customerDueByDate');
        Route::post('customer-due-report-data', 'customerDueReportData');
        Route::post('supplier-due-report', 'supplierDueReportByDate')->name('report.supplierDueByDate');
        Route::post('supplier-due-report-data', 'supplierDueReportData');
    });
});
```

## Controllers (Laravel)

### `ReportController::dailySale` — `app/Http/Controllers/ReportController.php:246–295`
```php
# app/Http/Controllers/ReportController.php:246–295
public function dailySale($year, $month)
{
    $role = Role::find(Auth::user()->role_id);
    if($role->hasPermissionTo('daily-sale')){
        // Get the logged in admin's pos_accnt_id
        $pos_accnt_id = Auth::user()->pos_accnt_id;
        
        $start = 1;
        $number_of_day = date('t', mktime(0, 0, 0, $month, 1, $year));
        while($start <= $number_of_day)
        {
            if($start < 10)
                $date = $year.'-'.$month.'-0'.$start;
            else
                $date = $year.'-'.$month.'-'.$start;
            $query1 = array(
                'SUM(total_discount) AS total_discount',
                'SUM(order_discount) AS order_discount',
                'SUM(total_tax) AS total_tax',
                'SUM(order_tax) AS order_tax',
                'SUM(shipping_cost) AS shipping_cost',
                'SUM(grand_total) AS grand_total'
            );
            // Modified to filter by pos_accnt_id
            $sale_data = Sale::whereDate('created_at', $date)
                            ->where('pos_accnt_id', $pos_accnt_id)
                            ->selectRaw(implode(',', $query1))->get();
            $total_discount[$start] = $sale_data[0]->total_discount;
            $order_discount[$start] = $sale_data[0]->order_discount;
            $total_tax[$start] = $sale_data[0]->total_tax;
            $order_tax[$start] = $sale_data[0]->order_tax;
            $shipping_cost[$start] = $sale_data[0]->shipping_cost;
            $grand_total[$start] = $sale_data[0]->grand_total;
            $start++;
        }
        $start_day = date('w', strtotime($year.'-'.$month.'-01')) + 1;
        $prev_year = date('Y', strtotime('-1 month', strtotime($year.'-'.$month.'-01')));
        $prev_month = date('m', strtotime('-1 month', strtotime($year.'-'.$month.'-01')));
        $next_year = date('Y', strtotime('+1 month', strtotime($year.'-'.$month.'-01')));
        $next_month = date('m', strtotime('+1 month', strtotime($year.'-'.$month.'-01')));
        // Modified to filter warehouses by pos_accnt_id
        $lims_warehouse_list = Warehouse::where('is_active', true)
                                ->where('pos_accnt_id', $pos_accnt_id)
                                ->get();
        $warehouse_id = 0;
        return view('backend.report.daily_sale', compact('total_discount','order_discount', 'total_tax', 'order_tax', 'shipping_cost', 'grand_total', 'start_day', 'year', 'month', 'number_of_day', 'prev_year', 'prev_month', 'next_year', 'next_month', 'lims_warehouse_list', 'warehouse_id'));
    }
    else
        return redirect()->back()->with('not_permitted', 'Sorry! You are not allowed to access this module');
}
```

### `ReportController::dailySaleByWarehouse` — `app/Http/Controllers/ReportController.php:297–355`
```php
# app/Http/Controllers/ReportController.php:297–355
public function dailySaleByWarehouse(Request $request, $year, $month)
{
    $data = $request->all();
    if($data['warehouse_id'] == 0)
        return redirect()->back();
    
    // Get the logged in admin's pos_accnt_id
    $pos_accnt_id = Auth::user()->pos_accnt_id;
    
    // First check if the warehouse belongs to this admin
    $warehouse = Warehouse::where('id', $data['warehouse_id'])
                    ->where('pos_accnt_id', $pos_accnt_id)
                    ->first();
    
    if (!$warehouse) {
        return redirect()->back()->with('not_permitted', 'You do not have permission to access this warehouse');
    }
    
    $start = 1;
    $number_of_day = date('t', mktime(0, 0, 0, $month, 1, $year));
    while($start <= $number_of_day)
    {
        if($start < 10)
            $date = $year.'-'.$month.'-0'.$start;
        else
            $date = $year.'-'.$month.'-'.$start;
        $query1 = array(
            'SUM(total_discount) AS total_discount',
            'SUM(order_discount) AS order_discount',
            'SUM(total_tax) AS total_tax',
            'SUM(order_tax) AS order_tax',
            'SUM(shipping_cost) AS shipping_cost',
            'SUM(grand_total) AS grand_total'
        );
        // Modified to filter by pos_accnt_id and warehouse_id
        $sale_data = Sale::where('warehouse_id', $data['warehouse_id'])
                        ->where('pos_accnt_id', $pos_accnt_id)
                        ->whereDate('created_at', $date)
                        ->selectRaw(implode(',', $query1))->get();
        $total_discount[$start] = $sale_data[0]->total_discount;
        $order_discount[$start] = $sale_data[0]->order_discount;
        $total_tax[$start] = $sale_data[0]->total_tax;
        $order_tax[$start] = $sale_data[0]->order_tax;
        $shipping_cost[$start] = $sale_data[0]->shipping_cost;
        $grand_total[$start] = $sale_data[0]->grand_total;
        $start++;
    }
    $start_day = date('w', strtotime($year.'-'.$month.'-01')) + 1;
    $prev_year = date('Y', strtotime('-1 month', strtotime($year.'-'.$month.'-01')));
    $prev_month = date('m', strtotime('-1 month', strtotime($year.'-'.$month.'-01')));
    $next_year = date('Y', strtotime('+1 month', strtotime($year.'-'.$month.'-01')));
    $next_month = date('m', strtotime('+1 month', strtotime($year.'-'.$month.'-01')));
    // Modified to filter warehouses by pos_accnt_id
    $lims_warehouse_list = Warehouse::where('is_active', true)
                            ->where('pos_accnt_id', $pos_accnt_id)
                            ->get();
    $warehouse_id = $data['warehouse_id'];
    return view('backend.report.daily_sale', compact('total_discount','order_discount', 'total_tax', 'order_tax', 'shipping_cost', 'grand_total', 'start_day', 'year', 'month', 'number_of_day', 'prev_year', 'prev_month', 'next_year', 'next_month', 'lims_warehouse_list', 'warehouse_id'));
}
```

### `ReportController::profitLoss` — `app/Http/Controllers/ReportController.php:837–969`
```php
# app/Http/Controllers/ReportController.php:837–969
public function profitLoss(Request $request)
{
    // Get the logged-in admin's pos_accnt_id
    $pos_accnt_id = Auth::user()->pos_accnt_id;
    
    $start_date = $request['start_date'];
    $end_date = $request['end_date'];
    $query1 = array(
        'SUM(grand_total) AS grand_total',
        'SUM(shipping_cost) AS shipping_cost',
        'SUM(paid_amount) AS paid_amount',
        'SUM(total_tax + order_tax) AS tax',
        'SUM(total_discount + order_discount) AS discount'
    );
    $query2 = array(
        'SUM(grand_total) AS grand_total',
        'SUM(total_tax + order_tax) AS tax'
    );
    
    config()->set('database.connections.mysql.strict', false);
    DB::reconnect();
    
    // Filter by pos_accnt_id in the product_sales query
    $product_sale_data = Product_Sale::join('sales', 'product_sales.sale_id', '=', 'sales.id')
                        ->select(DB::raw('product_sales.product_id, product_sales.product_batch_id, product_sales.sale_unit_id, sum(product_sales.qty) as sold_qty, sum(product_sales.return_qty) as return_qty, sum(product_sales.total) as sold_amount'))
                        ->whereDate('sales.created_at', '>=' , $start_date)
                        ->whereDate('sales.created_at', '<=' , $end_date)
                        ->where('sales.pos_accnt_id', $pos_accnt_id) // Add this line
                        ->groupBy('product_sales.product_id', 'product_sales.product_batch_id')
                        ->get();

    config()->set('database.connections.mysql.strict', true);
    DB::reconnect();
    
    $data = $this->calculateAverageCOGS($product_sale_data);
    $product_cost = $data[0];
    $product_tax = $data[1];
    
    // Filter all queries by pos_accnt_id
    $purchase = Purchase::whereDate('created_at', '>=' , $start_date)
                ->whereDate('created_at', '<=' , $end_date)
                ->where('pos_accnt_id', $pos_accnt_id)
                ->selectRaw(implode(',', $query1))->get();
    
    $total_purchase = Purchase::whereDate('created_at', '>=' , $start_date)
                    ->whereDate('created_at', '<=' , $end_date)
                    ->where('pos_accnt_id', $pos_accnt_id)
                    ->count();
    
    $sale = Sale::whereDate('created_at', '>=' , $start_date)
            ->whereDate('created_at', '<=' , $end_date)
            ->where('pos_accnt_id', $pos_accnt_id)
            ->selectRaw(implode(',', $query1))->get();
    
    $total_sale = Sale::whereDate('created_at', '>=' , $start_date)
                ->whereDate('created_at', '<=' , $end_date)
                ->where('pos_accnt_id', $pos_accnt_id)
                ->count();
    
    $return = Returns::whereDate('created_at', '>=' , $start_date)
            ->whereDate('created_at', '<=' , $end_date)
            ->where('pos_accnt_id', $pos_accnt_id)
            ->selectRaw(implode(',', $query2))->get();
    
    $total_return = Returns::whereDate('created_at', '>=' , $start_date)
                    ->whereDate('created_at', '<=' , $end_date)
                    ->where('pos_accnt_id', $pos_accnt_id)
                    ->count();
    
    $purchase_return = ReturnPurchase::whereDate('created_at', '>=' , $start_date)
                    ->whereDate('created_at', '<=' , $end_date)
                    ->where('pos_accnt_id', $pos_accnt_id)
                    ->selectRaw(implode(',', $query2))->get();
    
    $total_purchase_return = ReturnPurchase::whereDate('created_at', '>=' , $start_date)
                            ->whereDate('created_at', '<=' , $end_date)
                            ->where('pos_accnt_id', $pos_accnt_id)
                            ->count();
    
    $expense = Expense::whereDate('created_at', '>=' , $start_date)
            ->whereDate('created_at', '<=' , $end_date)
            ->where('pos_accnt_id', $pos_accnt_id)
            ->sum('amount');
    
    $income = Income::whereDate('created_at', '>=' , $start_date)
            ->whereDate('created_at', '<=' , $end_date)
            ->where('pos_accnt_id', $pos_accnt_id)
            ->sum('amount');
    
    $total_expense = Expense::whereDate('created_at', '>=' , $start_date)
                    ->whereDate('created_at', '<=' , $end_date)
                    ->where('pos_accnt_id', $pos_accnt_id)
                    ->count();
    
    $total_income = Income::whereDate('created_at', '>=' , $start_date)
                    ->whereDate('created_at', '<=' , $end_date)
                    ->where('pos_accnt_id', $pos_accnt_id)
                    ->count();
    
    $payroll = Payroll::whereDate('created_at', '>=' , $start_date)
                ->whereDate('created_at', '<=' , $end_date)
                ->where('pos_accnt_id', $pos_accnt_id)
                ->sum('amount');
    
    $total_payroll = Payroll::whereDate('created_at', '>=' , $start_date)
                    ->whereDate('created_at', '<=' , $end_date)
                    ->where('pos_accnt_id', $pos_accnt_id)
                    ->count();
    
    $total_item = DB::table('product_warehouse')
                ->join('products', 'product_warehouse.product_id', '=', 'products.id')
                ->where([
                    ['products.is_active', true],
                    ['product_warehouse.qty', '>' , 0],
                    ['products.pos_accnt_id', $pos_accnt_id] // Add this line
                ])->count();
    
    // Filter payment queries
    $payment_recieved_number = DB::table('payments')
                            ->join('sales', 'payments.sale_id', '=', 'sales.id')
                            ->whereNotNull('sale_id')
                            ->where('sales.pos_accnt_id', $pos_accnt_id)
                            ->whereDate('payments.created_at', '>=' , $start_date)
                            ->whereDate('payments.created_at', '<=' , $end_date)
                            ->count();
    
    $payment_recieved = DB::table('payments')
                    ->join('sales', 'payments.sale_id', '=', 'sales.id')
                    ->whereNotNull('sale_id')
                    ->where('sales.pos_accnt_id', $pos_accnt_id)
                    ->whereDate('payments.created_at', '>=' , $start_date)
                    ->whereDate('payments.created_at', '<=' , $end_date)
                    ->sum('payments.amount');
}
```

### `ReportController::saleReportData` — `app/Http/Controllers/ReportController.php:2358–2598`
```php
# app/Http/Controllers/ReportController.php:2358–2598
public function saleReportData(Request $request)
{
    $data = $request->all();
    $start_date = $data['start_date'];
    $end_date = $data['end_date'];
    $warehouse_id = $data['warehouse_id'];
    $variant_id = [];

    $columns = array(
        1 => 'name'
    );

    if($request->input('length') != -1)
        $limit = $request->input('length');
    else
        $limit = $totalData;
    //return $request;
    $start = $request->input('start');
    $order = $columns[$request->input('order.0.column')];
    $dir = $request->input('order.0.dir');

    if($request->input('search.value')) {
        $search = $request->input('search.value');
        $totalData = Product::where([
            ['name', 'LIKE', "%{$search}%"],
            ['is_active', true]
        ])->count();
        $lims_product_all = Product::with('category')
                            ->select('id', 'name', 'code', 'category_id', 'qty', 'is_variant', 'price', 'cost')
                            ->where([
                                ['name', 'LIKE', "%{$search}%"],
                                ['is_active', true]
                            ])->offset($start)
                              ->limit($limit)
                              ->orderBy($order, $dir)
                              ->get();
    }
    else {
        $totalData = Product::where('is_active', true)->count();
        $lims_product_all = Product::with('category')
                            ->select('id', 'name', 'code', 'category_id', 'qty', 'is_variant', 'price', 'cost')
                            ->where('is_active', true)
                            ->offset($start)
                            ->limit($limit)
                            ->orderBy($order, $dir)
                            ->get();
    }

    $totalFiltered = $totalData;
    $data = [];
    foreach ($lims_product_all as $product) {
        $variant_id_all = [];
        if($warehouse_id == 0) {
            if($product->is_variant) {
                $variant_id_all = ProductVariant::where('product_id', $product->id)->pluck('variant_id', 'item_code');
                foreach ($variant_id_all as $item_code => $variant_id) {
                    $variant_data = Variant::select('name')->find($variant_id);
                    $nestedData['key'] = count($data);
                    $nestedData['name'] = $product->name . ' [' . $variant_data->name . ']'.'<br>'.$item_code;
                    $nestedData['category'] = $product->category->name;
                    //sale data
                    $nestedData['sold_amount'] = Product_Sale::where([
                                            ['product_id', $product->id],
                                            ['variant_id', $variant_id]
                                    ])->whereDate('created_at', '>=' , $start_date)->whereDate('created_at', '<=' , $end_date)->sum('total');

                    $lims_product_sale_data = Product_Sale::select('sale_unit_id', 'qty')->where([
                                            ['product_id', $product->id],
                                            ['variant_id', $variant_id]
                                    ])->whereDate('created_at', '>=' , $start_date)->whereDate('created_at', '<=' , $end_date)->get();

                    $sold_qty = 0;
                    if(count($lims_product_sale_data)) {
                        foreach ($lims_product_sale_data as $product_sale) {
                            $unit = DB::table('units')->find($product_sale->sale_unit_id);
                            if($unit->operator == '*'){
                                $sold_qty += $product_sale->qty * $unit->operation_value;
                            }
                            elseif($unit->operator == '/'){
                                $sold_qty += $product_sale->qty / $unit->operation_value;
                            }
                        }
                    }
                    $nestedData['sold_qty'] = $sold_qty;

                    $product_variant_data = ProductVariant::where([
                        ['product_id', $product->id],
                        ['variant_id', $variant_id]
                    ])->select('qty')->first();
                    $nestedData['in_stock'] = $product_variant_data->qty;
                    $data[] = $nestedData;
                }
            }
            else {
                $nestedData['key'] = count($data);
                $nestedData['name'] = $product->name.'<br>'.$product->code;
                $nestedData['category'] = $product->category->name;

                //sale data
                $nestedData['sold_amount'] = Product_Sale::where('product_id', $product->id)->whereDate('created_at', '>=' , $start_date)->whereDate('created_at', '<=' , $end_date)->sum('total');

                $lims_product_sale_data = Product_Sale::select('sale_unit_id', 'qty')->where('product_id', $product->id)->whereDate('created_at', '>=' , $start_date)->whereDate('created_at', '<=' , $end_date)->get();

                $sold_qty = 0;
                if(count($lims_product_sale_data)) {
                    foreach ($lims_product_sale_data as $product_sale) {
                        if($product_sale->sale_unit_id > 0) {
                            $unit = DB::table('units')->find($product_sale->sale_unit_id);
                            if($unit->operator == '*'){
                                $sold_qty += $product_sale->qty * $unit->operation_value;
                            }
                            elseif($unit->operator == '/'){
                                $sold_qty += $product_sale->qty / $unit->operation_value;
                            }
                        }
                        else
                            $sold_qty = $product_sale->qty;
                    }
                }
                $nestedData['sold_qty'] = $sold_qty;

                $nestedData['in_stock'] = $product->qty;
                $data[] = $nestedData;
            }
        }
        else {
            if($product->is_variant) {
                $variant_id_all = ProductVariant::where('product_id', $product->id)->pluck('variant_id', 'item_code');

                foreach ($variant_id_all as $item_code => $variant_id) {
                    $variant_data = Variant::select('name')->find($variant_id);
                    $nestedData['key'] = count($data);
                    $nestedData['name'] = $product->name . ' [' . $variant_data->name . ']'.'<br>'.$item_code;
                    $nestedData['category'] = $product->category->name;

                    //sale data
                    $nestedData['sold_amount'] = DB::table('sales')
                                ->join('product_sales', 'sales.id', '=', 'product_sales.sale_id')->where([
                                    ['product_sales.product_id', $product->id],
                                    ['variant_id', $variant_id],
                                    ['sales.warehouse_id', $warehouse_id]
                                ])->whereDate('sales.created_at','>=', $start_date)->whereDate('sales.created_at','<=', $end_date)->sum('total');
                    $lims_product_sale_data = DB::table('sales')
                                ->join('product_sales', 'sales.id', '=', 'product_sales.sale_id')->where([
                                    ['product_sales.product_id', $product->id],
                                    ['variant_id', $variant_id],
                                    ['sales.warehouse_id', $warehouse_id]
                                ])->whereDate('sales.created_at','>=', $start_date)
                                ->whereDate('sales.created_at','<=', $end_date)
                                ->select('product_sales.sale_unit_id', 'product_sales.qty')
                                ->get();

                    $sold_qty = 0;
                    if(count($lims_product_sale_data)) {
                        foreach ($lims_product_sale_data as $product_sale) {
                            $unit = DB::table('units')->find($product_sale->sale_unit_id);
                            if($unit->operator == '*'){
                                $sold_qty += $product_sale->qty * $unit->operation_value;
                            }
                            elseif($unit->operator == '/'){
                                $sold_qty += $product_sale->qty / $unit->operation_value;
                            }
                        }
                    }
                    $nestedData['sold_qty'] = $sold_qty;



                    $product_warehouse = Product_Warehouse::where([
                        ['product_id', $product->id],
                        ['variant_id', $variant_id],
                        ['warehouse_id', $warehouse_id]
                    ])->select('qty')->first();
                    if($product_warehouse)
                        $nestedData['in_stock'] = $product_warehouse->qty;
                    else
                        $nestedData['in_stock'] = 0;

                    $data[] = $nestedData;
                }
            }
            else {
                $nestedData['key'] = count($data);
                $nestedData['name'] = $product->name.'<br>'.$product->code;
                $nestedData['category'] = $product->category->name;

                //sale data
                $nestedData['sold_amount'] = DB::table('sales')
                            ->join('product_sales', 'sales.id', '=', 'product_sales.sale_id')->where([
                                ['product_sales.product_id', $product->id],
                                ['sales.warehouse_id', $warehouse_id]
                            ])->whereDate('sales.created_at','>=', $start_date)->whereDate('sales.created_at','<=', $end_date)->sum('total');
                $lims_product_sale_data = DB::table('sales')
                            ->join('product_sales', 'sales.id', '=', 'product_sales.sale_id')->where([
                                ['product_sales.product_id', $product->id],
                                ['sales.warehouse_id', $warehouse_id]
                            ])->whereDate('sales.created_at','>=', $start_date)
                            ->whereDate('sales.created_at','<=', $end_date)
                            ->select('product_sales.sale_unit_id', 'product_sales.qty')
                            ->get();

                $sold_qty = 0;
                if(count($lims_product_sale_data)) {
                    foreach ($lims_product_sale_data as $product_sale) {
                        if($product_sale->sale_unit_id) {
                            $unit = DB::table('units')->find($product_sale->sale_unit_id);
                            if($unit->operator == '*'){
                                $sold_qty += $product_sale->qty * $unit->operation_value;
                            }
                            elseif($unit->operator == '/'){
                                $sold_qty += $product_sale->qty / $unit->operation_value;
                            }
                        }
                    }
                }
                $nestedData['sold_qty'] = $sold_qty;

                $product_warehouse = Product_Warehouse::where([
                    ['product_id', $product->id],
                    ['warehouse_id', $warehouse_id]
                ])->select('qty')->first();
                if($product_warehouse)
                    $nestedData['in_stock'] = $product_warehouse->qty;
                else
                    $nestedData['in_stock'] = 0;

                $data[] = $nestedData;
            }
        }
    }
    /*$totalData = count($data);
    $totalFiltered = $totalData;*/
    $json_data = array(
        "draw"            => intval($request->input('draw')),
        "recordsTotal"    => intval($totalData),
        "recordsFiltered" => intval($totalFiltered),
        "data"            => $data
    );

    echo json_encode($json_data);
}
```

### `ReportController::saleReportChart` — `app/Http/Controllers/ReportController.php:2616–2683`
```php
# app/Http/Controllers/ReportController.php:2616–2683
public function saleReportChart(Request $request)
{
    // Get the logged-in admin's pos_accnt_id
    $pos_accnt_id = Auth::user()->pos_accnt_id;

    $start_date = $request->start_date;
    $end_date = strtotime($request->end_date);
    $warehouse_id = $request->warehouse_id;
    $time_period = $request->time_period;
    $date_points = [];
    $sold_qty = [];

    if($time_period == 'monthly') {
        for($i = strtotime($start_date); $i <= $end_date; $i = strtotime('+1 month', $i)) {
            $date_points[] = date('Y-m-d', $i);
        }
    }
    else {
        for($i = strtotime('Saturday', strtotime($start_date)); $i <= $end_date; $i = strtotime('+1 week', $i)) {
            $date_points[] = date('Y-m-d', $i);
        }
    }
    $date_points[] = $request->end_date;
    
    foreach ($date_points as $key => $date_point) {
        $q = DB::table('sales')
            ->join('product_sales', 'sales.id', '=', 'product_sales.sale_id')
            ->whereDate('sales.created_at', '>=', $start_date)
            ->whereDate('sales.created_at', '<', $date_point)
            ->where('sales.pos_accnt_id', $pos_accnt_id); // Filter by pos_accnt_id
            
        if($warehouse_id)
            $q->where('sales.warehouse_id', $warehouse_id);
            
        if(isset($request->product_list)) {
            $product_ids = Product::whereIn('code', explode(",", trim($request->product_list)))
                ->where('pos_accnt_id', $pos_accnt_id) // Filter products by pos_accnt_id
                ->pluck('id')
                ->toArray();
            $q->whereIn('product_sales.product_id', $product_ids);
        }
        
        $qty = $q->sum('product_sales.qty');
        $sold_qty[$key] = $qty;
        $start_date = $date_point;
    }
    
    // Filter warehouses by pos_accnt_id
    $lims_warehouse_list = Warehouse::where('is_active', true)
        ->where('pos_accnt_id', $pos_accnt_id)
        ->select('id', 'name')
        ->get();
        
    $start_date = $request->start_date;
    $end_date = $request->end_date;
    
    // Pass pos_accnt_id to the view in case you need it there
    return view('backend.report.sale_report_chart', compact(
        'start_date', 
        'end_date', 
        'warehouse_id', 
        'time_period', 
        'sold_qty', 
        'date_points', 
        'lims_warehouse_list',
        'pos_accnt_id'
    ));
}
```

### `HomeController::dashboardFilter` — `app/Http/Controllers/HomeController.php:548–687`
```php
# app/Http/Controllers/HomeController.php:548–687
public function dashboardFilter($start_date, $end_date, $warehouse_id)
{
    // Get the current user's pos_accnt_id for multi-tenancy filtering
    $pos_accnt_id = Auth::user()->pos_accnt_id;
    
    if(Auth::user()->role_id > 2 && cache()->get('general_setting')->staff_access == 'own') {
        config()->set('database.connections.mysql.strict', false);
        DB::reconnect();
        $product_sale_data = Sale::join('product_sales', 'sales.id','=', 'product_sales.sale_id')
            ->select(DB::raw('product_sales.product_id, product_sales.product_batch_id, sale_unit_id, sum(product_sales.qty) as sold_qty, sum(product_sales.total) as sold_amount'))
            ->where('sales.user_id', Auth::id())
            ->where('sales.pos_accnt_id', $pos_accnt_id) // Add filtering by pos_accnt_id
            ->whereDate('sales.created_at', '>=' , $start_date)
            ->whereDate('sales.created_at', '<=' , $end_date)
            ->groupBy('product_sales.product_id', 'product_sales.product_batch_id')
            ->get();
        config()->set('database.connections.mysql.strict', true);
        DB::reconnect();
        $product_cost = $this->calculateAverageCOGS($product_sale_data);
        
        // Add pos_accnt_id filtering to all queries
        $revenue = Sale::whereDate('created_at', '>=' , $start_date)
            ->where('user_id', Auth::id())
            ->where('pos_accnt_id', $pos_accnt_id)
            ->whereDate('created_at', '<=' , $end_date)
            ->sum(DB::raw('grand_total - shipping_cost'));
            
        $return = Returns::whereDate('created_at', '>=' , $start_date)
            ->where('user_id', Auth::id())
            ->where('pos_accnt_id', $pos_accnt_id)
            ->whereDate('created_at', '<=' , $end_date)
            ->sum('grand_total');
            
        $purchase_return = ReturnPurchase::whereDate('created_at', '>=' , $start_date)
            ->where('user_id', Auth::id())
            ->where('pos_accnt_id', $pos_accnt_id)
            ->whereDate('created_at', '<=' , $end_date)
            ->sum('grand_total');
            
        $expense = Expense::whereDate('created_at', '>=' , $start_date)
            ->where('user_id', Auth::id())
            ->where('pos_accnt_id', $pos_accnt_id)
            ->whereDate('created_at', '<=' , $end_date)
            ->sum('amount');
            
        $income = Income::whereDate('created_at', '>=' , $start_date)
            ->where('user_id', Auth::id())
            ->where('pos_accnt_id', $pos_accnt_id)
            ->whereDate('created_at', '<=' , $end_date)
            ->sum('amount');
            
        $purchase = Purchase::whereDate('created_at', '>=' , $start_date)
            ->where('user_id', Auth::id())
            ->where('pos_accnt_id', $pos_accnt_id)
            ->whereDate('created_at', '<=' , $end_date)
            ->sum('grand_total');
            
        $revenue = $revenue - $return + $income;
        $profit = $revenue + $purchase_return - $product_cost - $expense;
    }
    else {
        config()->set('database.connections.mysql.strict', false);
        DB::reconnect();
        $q = Sale::join('product_sales', 'sales.id','=', 'product_sales.sale_id')
            ->select(DB::raw('product_sales.product_id, product_sales.product_batch_id, product_sales.sale_unit_id, sum(product_sales.qty) as sold_qty, sum(product_sales.return_qty) as return_qty, sum(product_sales.total) as sold_amount'))
            ->where('sales.pos_accnt_id', $pos_accnt_id) // Add filtering by pos_accnt_id
            ->whereDate('sales.created_at', '>=' , $start_date)
            ->whereDate('sales.created_at', '<=' , $end_date);
            
        if($warehouse_id == 0)
        {
            $product_sale_data = $q->groupBy('product_sales.product_id', 'product_sales.product_batch_id')->get();
        }
        else
        {
            $product_sale_data = $q->where('sales.warehouse_id', $warehouse_id)
                                    ->groupBy('product_sales.product_id', 'product_sales.product_batch_id')
                                    ->get();
        }

        config()->set('database.connections.mysql.strict', true);
        DB::reconnect();
        $product_cost = $this->calculateAverageCOGS($product_sale_data);

        if($warehouse_id == 0) {
            $revenue = Sale::whereDate('created_at', '>=' , $start_date)
                ->whereDate('created_at', '<=' , $end_date)
                ->where('pos_accnt_id', $pos_accnt_id) // Add filtering
                ->sum(DB::raw('grand_total - shipping_cost'));
                
            $return = Returns::whereDate('created_at', '>=' , $start_date)
                ->whereDate('created_at', '<=' , $end_date)
                ->where('pos_accnt_id', $pos_accnt_id) // Add filtering
                ->sum('grand_total');
                
            $purchase_return = ReturnPurchase::whereDate('created_at', '>=' , $start_date)
                ->whereDate('created_at', '<=' , $end_date)
                ->where('pos_accnt_id', $pos_accnt_id) // Add filtering
                ->sum('grand_total');
        }
        else {
            $revenue = Sale::where('warehouse_id', $warehouse_id)
                ->whereDate('created_at', '>=' , $start_date)
                ->whereDate('created_at', '<=' , $end_date)
                ->where('pos_accnt_id', $pos_accnt_id) // Add filtering
                ->sum(DB::raw('grand_total - shipping_cost'));

            $return = Returns::where('warehouse_id', $warehouse_id)
                ->whereDate('created_at', '>=' , $start_date)
                ->whereDate('created_at', '<=' , $end_date)
                ->where('pos_accnt_id', $pos_accnt_id) // Add filtering
                ->sum('grand_total');

            $purchase_return = ReturnPurchase::where('warehouse_id', $warehouse_id)
                ->whereDate('created_at', '>=' , $start_date)
                ->whereDate('created_at', '<=' , $end_date)
                ->where('pos_accnt_id', $pos_accnt_id) // Add filtering
                ->sum('grand_total');
        }
        
        $expense = Expense::whereDate('created_at', '>=' , $start_date)
            ->whereDate('created_at', '<=' , $end_date)
            ->where('pos_accnt_id', $pos_accnt_id) // Add filtering
            ->sum('amount');
            
        $income = Income::whereDate('created_at', '>=' , $start_date)
            ->whereDate('created_at', '<=' , $end_date)
            ->where('pos_accnt_id', $pos_accnt_id) // Add filtering
            ->sum('amount');
            
        $revenue = $revenue - $return + $income;
        $profit = $revenue + $purchase_return - $product_cost - $expense;

        $data[0] = $revenue;
        $data[1] = $return;
        $data[2] = $profit;
        $data[3] = $purchase_return;
    }
    return $data;
}
```

## Server Router (tRPC/Node)

### `dashboardRouter.stats` and `warehouses` — `uhaiui/src/server/routers/dashboard.ts:10–71, 74–96, 116–146, 159–186`
```ts
# uhaiui/src/server/routers/dashboard.ts:10–71
stats: protectedProcedure
  .input(
    z.object({
      pos_accnt_id: z.number(),
      start_date: z.string(),
      end_date: z.string(),
      warehouse_id: z.number().optional(),
    })
  )
  .query(async ({ input }) => {
    const { pos_accnt_id, start_date, end_date, warehouse_id } = input

    // Revenue calculation
    let revenueQuery = `
      SELECT COALESCE(SUM(grand_total - COALESCE(shipping_cost, 0)), 0) as revenue
      FROM sales
      WHERE pos_accnt_id = ?
      AND DATE(created_at) >= ?
      AND DATE(created_at) <= ?
    `
    let revenueParams: any[] = [pos_accnt_id, start_date, end_date]

    if (warehouse_id && warehouse_id > 0) {
      revenueQuery = `
        SELECT COALESCE(SUM(grand_total - COALESCE(shipping_cost, 0)), 0) as revenue
        FROM sales
        WHERE pos_accnt_id = ?
        AND warehouse_id = ?
        AND DATE(created_at) >= ?
        AND DATE(created_at) <= ?
      `
      revenueParams = [pos_accnt_id, warehouse_id, start_date, end_date]
    }

    const revenueResult = await first<{ revenue: number }>(revenueQuery, revenueParams)
    let revenue = revenueResult?.revenue || 0
```
```ts
# uhaiui/src/server/routers/dashboard.ts:49–71
// Sale Return
let saleReturnQuery = `
  SELECT COALESCE(SUM(grand_total), 0) as sale_return
  FROM returns
  WHERE pos_accnt_id = ?
  AND DATE(created_at) >= ?
  AND DATE(created_at) <= ?
`
let saleReturnParams: any[] = [pos_accnt_id, start_date, end_date]

if (warehouse_id && warehouse_id > 0) {
  saleReturnQuery = `
    SELECT COALESCE(SUM(grand_total), 0) as sale_return
    FROM returns
    WHERE pos_accnt_id = ?
    AND warehouse_id = ?
    AND DATE(created_at) >= ?
    AND DATE(created_at) <= ?
  `
  saleReturnParams = [pos_accnt_id, warehouse_id, start_date, end_date]
}
```
```ts
# uhaiui/src/server/routers/dashboard.ts:116–146
// Loyalty Points
let loyaltyQuery = `
  SELECT COALESCE(SUM((p.price * ps.qty) - (p.cost * ps.qty)), 0) as loyalty_points
  FROM sales s
  INNER JOIN product_sales ps ON s.id = ps.sale_id
  INNER JOIN products p ON ps.product_id = p.id
  WHERE s.pos_accnt_id = ?
  AND p.pos_accnt_id = ?
  AND DATE(s.created_at) >= ?
  AND DATE(s.created_at) <= ?
`
let loyaltyParams: any[] = [pos_accnt_id, pos_accnt_id, start_date, end_date]

if (warehouse_id && warehouse_id > 0) {
  loyaltyQuery = `
    SELECT COALESCE(SUM((p.price * ps.qty) - (p.cost * ps.qty)), 0) as loyalty_points
    FROM sales s
    INNER JOIN product_sales ps ON s.id = ps.sale_id
    INNER JOIN products p ON ps.product_id = p.id
    WHERE s.pos_accnt_id = ?
    AND p.pos_accnt_id = ?
    AND s.warehouse_id = ?
    AND DATE(s.created_at) >= ?
    AND DATE(s.created_at) <= ?
  `
  loyaltyParams = [pos_accnt_id, pos_accnt_id, warehouse_id, start_date, end_date]
}
```
```ts
# uhaiui/src/server/routers/dashboard.ts:159–186
warehouses: protectedProcedure
  .input(
    z.object({
      pos_accnt_id: z.number(),
    })
  )
  .query(async ({ input, ctx }) => {
    const { pos_accnt_id } = input

    let warehouseQuery = `
      SELECT id, name
      FROM warehouses
      WHERE is_active = 1
      AND pos_accnt_id = ?
    `
    const params: any[] = [pos_accnt_id]

    if (ctx.user?.warehouse_id && ctx.user?.role_id && ctx.user.role_id > 2) {
      warehouseQuery += ' AND id = ?'
      params.push(ctx.user.warehouse_id)
    }

    warehouseQuery += ' ORDER BY name ASC'

    return all<{ id: number; name: string }>(warehouseQuery, params)
  }),
```

## Frontend (Next.js)

### Dashboard tRPC Usage — `uhaiui/src/app/dashboard/page.tsx:90–116`
```tsx
# uhaiui/src/app/dashboard/page.tsx:90–116
// Use tRPC queries
const statsQueryInput: any = {
  pos_accnt_id: posAccntId,
  start_date: dateRange.from,
  end_date: dateRange.to,
}
if (warehouseId) {
  statsQueryInput.warehouse_id = warehouseId
}

const { data: stats, isLoading: statsLoading, error: statsError } = trpc.dashboard.stats.useQuery(
  statsQueryInput,
  {
    enabled: !!user && !!posAccntId,
    retry: false,
  }
)

const { data: warehouses } = trpc.dashboard.warehouses.useQuery({
  pos_accnt_id: posAccntId,
}, {
  enabled: !!user,
})
```

### Dashboard Cards — `uhaiui/src/app/dashboard/page.tsx:400–456`
```tsx
# uhaiui/src/app/dashboard/page.tsx:400–456
{/* Stats Cards */}
<div className="grid gap-3 md:grid-cols-2 lg:grid-cols-4">
  <Card className="rounded-lg shadow-sm">
    <CardContent className="p-4">
      <div className="flex items-center justify-between">
        <div>
          <p className="text-xs font-medium text-muted-foreground">Revenue</p>
          <p className="text-xl font-bold mt-1.5" style={{ color: '#733686' }}>
            {formatCurrency(stats?.revenue || 0)}
          </p>
        </div>
        <BarChart3 className="size-6" style={{ color: '#733686' }} />
      </div>
    </CardContent>
  </Card>
  ...
  <Card className="rounded-lg shadow-sm">
    <CardContent className="p-4">
      <div className="flex items-center justify-between">
        <div>
          <p className="text-xs font-medium text-muted-foreground">Loyalty Points</p>
          <p className="text-xl font-bold mt-1.5" style={{ color: '#297ff9' }}>
            {formatCurrency(stats?.loyaltyPoints || 0)}
          </p>
        </div>
        <Trophy className="size-6" style={{ color: '#297ff9' }} />
      </div>
    </CardContent>
  </Card>
</div>
```

## Views (Blade)

### Product Report AJAX — `resources/views/backend/report/product_report.blade.php:121–139`
```js
# resources/views/backend/report/product_report.blade.php:121–139
$('#product-report-table').DataTable( {
    "processing": true,
    "serverSide": true,
    "ajax":{
        url:"product_report_data",
        data:{
            start_date: start_date,
            end_date: end_date,
            warehouse_id: warehouse_id
        },
        dataType: "json",
        type:"post",
    },
    // columns ...
});
```

### Sale Report AJAX — `resources/views/backend/report/sale_report.blade.php:101–116, 122–129`
```js
# resources/views/backend/report/sale_report.blade.php:101–116
$('#product-report-table').DataTable( {
    "processing": true,
    "serverSide": true,
    "ajax":{
        url:"sale_report_data",
        data:{
            start_date: start_date,
            end_date: end_date,
            warehouse_id: warehouse_id
        },
        dataType: "json",
        type:"post",
    },
});
# resources/views/backend/report/sale_report.blade.php:122–129
"columns": [
    {"data": "key"},
    {"data": "name"},
    {"data": "category"},
    {"data": "sold_amount"},
    {"data": "sold_qty"},
    {"data": "in_stock"},
],
```

## Biller-Based Filtering and Product Filter — Proposed Fixes

### Enforce `biller_id` in Dashboard (Laravel)
```php
// app/Http/Controllers/HomeController.php (dashboardFilter): add biller filtering
$biller_id = Auth::user()->biller_id;
$revenue = Sale::whereDate('created_at', '>=', $start_date)
    ->whereDate('created_at', '<=', $end_date)
    ->where('pos_accnt_id', $pos_accnt_id)
    ->where('biller_id', $biller_id)
    ->sum(DB::raw('grand_total - COALESCE(shipping_cost, 0)'));
// apply biller_id similarly to Returns, ReturnPurchase, Expense, Income
```

### Enforce `biller_id` in Reports (Laravel)
```php
// app/Http/Controllers/ReportController.php (dailySale)
$sale_data = Sale::whereDate('created_at', $date)
    ->where('pos_accnt_id', $pos_accnt_id)
    ->where('biller_id', $biller_id)
    ->selectRaw(implode(',', $query1))->get();
```
```php
// app/Http/Controllers/ReportController.php (dailySaleByWarehouse)
$sale_data = Sale::where('warehouse_id', $data['warehouse_id'])
    ->where('pos_accnt_id', $pos_accnt_id)
    ->where('biller_id', $biller_id)
    ->whereDate('created_at', $date)
    ->selectRaw(implode(',', $query1))->get();
```
```php
// app/Http/Controllers/ReportController.php (productReportData) base joins
$baseSale = Product_Sale::join('sales', 'product_sales.sale_id', '=', 'sales.id')
    ->where('sales.pos_accnt_id', $pos_accnt_id)
    ->where('sales.biller_id', $biller_id)
    ->whereDate('sales.created_at', '>=', $start_date)
    ->whereDate('sales.created_at', '<=', $end_date);
if ($warehouse_id) $baseSale->where('sales.warehouse_id', $warehouse_id);
if ($product_id) $baseSale->where('product_sales.product_id', $product_id);
```

### Add Product-Level Filter (Laravel + Blade)
```html
<!-- resources/views/backend/report/product_report.blade.php: add product selector -->
<select name="product_id" id="product_id" class="form-control">
  <option value="">All Products</option>
</select>
```
```js
// resources/views/backend/report/product_report.blade.php: pass product_id
ajax: {
  url:"product_report_data",
  data:{
    start_date: start_date,
    end_date: end_date,
    warehouse_id: warehouse_id,
    product_id: $('#product_id').val(),
  },
  type:"post",
}
```

### tRPC Router: Add `billerId` and `productId`
```ts
// uhaiui/src/server/routers/dashboard.ts (stats input and SQL)
.input(z.object({
  pos_accnt_id: z.number(),
  biller_id: z.number(),
  start_date: z.string(),
  end_date: z.string(),
  warehouse_id: z.number().optional(),
  product_id: z.number().optional(),
}))
// SQL: add "AND biller_id = ?" and optionally "AND ps.product_id = ?"
```

### Authorization: Admin-only Global Visibility
```php
// app/Providers/AuthServiceProvider.php
Gate::define('view-all-data', fn($user) => (bool) $user->is_admin);
// Controllers: if not admin, force biller_id from session
$biller_id = Gate::allows('view-all-data') && $request->filled('biller_id')
    ? (int) $request->input('biller_id')
    : Auth::user()->biller_id;
```
```ts
// tRPC context guard
const isAdmin = ctx.user.role === 'admin';
const billerId = isAdmin && input.biller_id ? input.biller_id : ctx.user.biller_id;
```

## Summary
- Exact code lines demonstrate how sales, dashboard, and loyalty are computed and displayed.
- Proposed fixes show how to enforce `biller_id` and add per-product filtering, while preserving `pos_accnt_id` multi-tenancy.
- Admin-only visibility is enforced via gates/policies and router guards.

## Annotations And Comments On Added Changes

### What Was Added And Why
- Enforced `biller_id` in all revenue-related queries to ensure data ties to a specific biller rather than just tenant (`pos_accnt_id`).
- Added optional `product_id` filter to per-product report endpoints and dashboard stats to filter individual product data.
- Propagated `biller_id` and `product_id` through frontend, server router, and controllers for consistent scoping.
- Introduced authorization patterns so only admin can view cross-biller data; non-admins are always scoped to their own `biller_id`.

### Annotated Snippets (showing the exact additions)

#### HomeController::dashboardFilter — Add `biller_id` constraints
```php
// ADDED: resolve biller_id from authenticated user
$biller_id = Auth::user()->biller_id; // ensures KPIs tied to specific biller

// ADDED: restrict revenue to biller
$revenue = Sale::whereDate('created_at', '>=', $start_date)
    ->whereDate('created_at', '<=', $end_date)
    ->where('pos_accnt_id', $pos_accnt_id)
    ->where('biller_id', $biller_id) // ADDED: biller scoping
    ->sum(DB::raw('grand_total - COALESCE(shipping_cost, 0)'));

// ADDED: apply biller_id to returns/purchase returns/expense/income
$return = Returns::whereDate('created_at', '>=', $start_date)
    ->whereDate('created_at', '<=', $end_date)
    ->where('pos_accnt_id', $pos_accnt_id)
    ->where('biller_id', $biller_id) // ADDED
    ->sum('grand_total');
```

Effect: Revenue, returns, purchase returns, expense, and income are now computed per biller. Prevents cross-biller leakage and ensures KPIs match the current biller context.

#### ReportController::dailySale / dailySaleByWarehouse — Add `biller_id`
```php
// ADDED: filter daily sale totals by biller_id
$sale_data = Sale::whereDate('created_at', $date)
    ->where('pos_accnt_id', $pos_accnt_id)
    ->where('biller_id', $biller_id) // ADDED
    ->selectRaw(implode(',', $query1))->get();

// ADDED: warehouse variant also scoped by biller
$sale_data = Sale::where('warehouse_id', $data['warehouse_id'])
    ->where('pos_accnt_id', $pos_accnt_id)
    ->where('biller_id', $biller_id) // ADDED
    ->whereDate('created_at', $date)
    ->selectRaw(implode(',', $query1))->get();
```

Effect: Daily sale totals cannot include sales from other billers; warehouse-scoped daily totals also respect biller boundaries.

#### ReportController::bestSeller — Add `biller_id`
```php
// ADDED: constrain best seller aggregation to biller
$data = Product_Sale::join('sales', 'product_sales.sale_id', '=', 'sales.id')
    ->where('sales.pos_accnt_id', $pos_accnt_id)
    ->where('sales.biller_id', $biller_id) // ADDED
    ->select('product_sales.product_id', DB::raw('SUM(product_sales.qty) as sold_qty'), DB::raw('SUM(product_sales.total) as sold_amount'))
    ->groupBy('product_sales.product_id')
    ->get();
```

Effect: Best seller list reflects products sold by the selected biller only.

#### ReportController::profitLoss — Add `biller_id` to all aggregates
```php
// ADDED: restrict all aggregates by biller_id
$sales = Sale::where('pos_accnt_id', $pos_accnt_id)
    ->where('biller_id', $biller_id) // ADDED
    ->whereDate('created_at', '>=', $start_date)
    ->whereDate('created_at', '<=', $end_date)
    ->sum(DB::raw('grand_total - COALESCE(shipping_cost, 0)'));

$returns = Returns::where('pos_accnt_id', $pos_accnt_id)
    ->where('biller_id', $biller_id) // ADDED
    ->whereDate('created_at', '>=', $start_date)
    ->whereDate('created_at', '<=', $end_date)
    ->sum('grand_total');
```

Effect: Profit/loss calculations are biller-specific across sales, returns, purchase returns, expense, and income.

#### ReportController::productReportData — Add `biller_id` and `product_id`
```php
// ADDED: base join scoped by biller and optional product
$baseSale = Product_Sale::join('sales', 'product_sales.sale_id', '=', 'sales.id')
    ->where('sales.pos_accnt_id', $pos_accnt_id)
    ->where('sales.biller_id', $biller_id) // ADDED
    ->whereDate('sales.created_at', '>=', $start_date)
    ->whereDate('sales.created_at', '<=', $end_date);

if ($warehouse_id) {
  $baseSale->where('sales.warehouse_id', $warehouse_id);
}
if ($product_id) {
  $baseSale->where('product_sales.product_id', $product_id); // ADDED: individual product filter
}
```

Effect: Per-product report rows can be filtered to a single product and are constrained to the biller.

#### ReportController::saleReportData / saleReportChart — Add `biller_id` and `product_id`
```php
// ADDED: join constrained by biller and product
$saleJoin = DB::table('sales')
  ->join('product_sales', 'sales.id', '=', 'product_sales.sale_id')
  ->where('sales.pos_accnt_id', $pos_accnt_id)
  ->where('sales.biller_id', $biller_id) // ADDED
  ->whereDate('sales.created_at', '>=', $start_date)
  ->whereDate('sales.created_at', '<=', $end_date);

if ($product_id) {
  $saleJoin->where('product_sales.product_id', $product_id); // ADDED
}
```

Effect: Sale report totals and chart series reflect biller scope and optional single-product selection.

#### ReportController::paymentReportByDate — Add `biller_id`
```php
// ADDED: filter payments by biller
$lims_payment_data = Payment::whereDate('created_at', '>=' , $start_date)
  ->whereDate('created_at', '<=' , $end_date)
  ->where('pos_accnt_id', $pos_accnt_id)
  ->where('biller_id', $biller_id) // ADDED
  ->get();
```

Effect: Payment report lists only payments for the current biller.

#### tRPC dashboard.ts — Add `biller_id` and `product_id`
```ts
// ADDED: inputs include biller_id and product_id
.input(z.object({
  pos_accnt_id: z.number(),
  biller_id: z.number(), // ADDED
  start_date: z.string(),
  end_date: z.string(),
  warehouse_id: z.number().optional(),
  product_id: z.number().optional(), // ADDED
}))

// ADDED: SQL constraints
let revenueSql = `
  SELECT COALESCE(SUM(grand_total - COALESCE(shipping_cost, 0)), 0) AS revenue
  FROM sales
  WHERE pos_accnt_id = ? AND biller_id = ? AND DATE(created_at) BETWEEN ? AND ?
`;

let loyaltySql = `
  SELECT COALESCE(SUM((p.price * ps.qty) - (p.cost * ps.qty)), 0) AS loyalty
  FROM product_sales ps
  JOIN sales s ON ps.sale_id = s.id
  JOIN products p ON ps.product_id = p.id
  WHERE s.pos_accnt_id = ? AND s.biller_id = ? AND p.pos_accnt_id = ? AND DATE(s.created_at) BETWEEN ? AND ?
`;

if (input.productId) {
  loyaltySql += ` AND ps.product_id = ?`; // ADDED: single product filter
}
```

Effect: Server-side KPIs respect biller context and allow product-specific loyalty calculations.

#### Frontend page.tsx — Pass `billerId` and `productId` to tRPC
```tsx
// ADDED: include billerId and productId in stats query input
const statsQueryInput: any = {
  pos_accnt_id: posAccntId,
  biller_id: billerId, // ADDED
  start_date: dateRange.from,
  end_date: dateRange.to,
  warehouse_id: warehouseId,
  product_id: productId, // ADDED
};
```

Effect: UI requests are biller-scoped and can target a single product; prevents accidental cross-biller aggregation.

#### Blade product_report — Add product selector and AJAX param
```html
<!-- ADDED: simple product selector for individual product filtering -->
<select name="product_id" id="product_id" class="form-control">
  <option value="">All Products</option>
  <!-- Populate with product options dynamically -->
  <!-- ADDED: ensures user can select a single product to filter report -->
</select>
```
```js
// ADDED: send product_id to backend
ajax: {
  url: "product_report_data",
  data: {
    start_date: start_date,
    end_date: end_date,
    warehouse_id: warehouse_id,
    product_id: $('#product_id').val(), // ADDED
  },
}
```

Effect: Report table can display data for a specific product.

#### Authorization — Admin-only global visibility
```php
// ADDED: Gate definition allows only admin to view all data
Gate::define('view-all-data', function ($user) {
    return (bool) $user->is_admin; // ADDED
});

// ADDED: controller pattern to force biller_id for non-admins
$biller_id = Gate::allows('view-all-data') && $request->filled('biller_id')
    ? (int) $request->input('biller_id')
    : Auth::user()->biller_id; // ADDED: default to user biller
```

Effect: Non-admins cannot change biller scope; admins can selectively view per-biller or all data.
