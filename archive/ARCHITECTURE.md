# ARCHITECTURE.md

# Mobile Shop ERP Architecture

## Philosophy

The application extends ERPNext instead of replacing it. Reuse built-in
ERP features wherever possible and keep all custom business logic inside
the `mobile_shop` app.

## Built-in ERPNext DocTypes

-   Supplier
-   Customer
-   User
-   Role
-   Address
-   Contact
-   Print Format
-   Report

These must never be modified directly.

## Custom DocTypes

### Mobile Shop Settings (Singleton)

Stores: - VAT Rate - Business Name - Invoice Footer - Warranty Defaults

### Phone

Represents one physical second-hand phone.

Fields include: - IMEI - Brand - Model - Purchase Price - Supplier -
Status - Condition - Battery Health

### Purchase Entry

Represents acquiring a phone.

Responsibilities: - Validate IMEI uniqueness - Create Phone record -
Mark Phone as In Stock

### Sales Entry

Represents selling a phone.

Responsibilities: - Validate Phone availability - Calculate Bahrain PMS
values - Mark Phone Sold - Store internal margin, VAT and net profit

## Relationships

Supplier (ERPNext) \| +--\> Purchase Entry \| +--\> Phone \| +--\> Sales
Entry \| +--\> Customer (ERPNext)

## Bahrain VAT

All VAT calculations must be server-side.

Margin = Selling Price - Purchase Price

If Margin \<= 0: - Margin = 0 - VAT = 0

Else: VAT = Margin × VAT Rate / (100 + VAT Rate)

Net Profit = Margin - VAT

VAT rate comes from Mobile Shop Settings.

## Folder Structure

mobile_shop/ ├── doctype/ │ ├── mobile_shop_settings/ │ ├── phone/ │ ├──
purchase_entry/ │ └── sales_entry/ ├── dashboard/ ├── report/ ├──
print_format/ ├── permissions/ ├── api.py ├── utils.py └── hooks.py

## Future Expansion

-   ERPNext Stock integration
-   Serial Number integration
-   Purchase Receipt integration
-   Sales Invoice integration
-   Repair tracking
-   Multi-branch
-   Warranty
