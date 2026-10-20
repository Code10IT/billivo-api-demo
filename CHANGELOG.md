# Changelog

All notable changes to the Billivo API are in this file.

## 1.4.1 (2026-10-20)

### Changed
- Suppliers: the EU VAT prefix is ignored to compare tax ids, so `ES74894836M` and `74894836M` are the same supplier.

## 1.4.0 (2026-10-20)

### Added
- `GET /expenses/suppliers`: list the suppliers of the expenses.
- `PATCH /expenses/suppliers`: change a supplier, for example to save the ERP id in `erpId`.

## 1.3.2 (2026-10-19)

### Added
- `dueDate`, `iban`, `bic` and `bankName` fields on expenses (payment due date and merchant bank account for a transfer).

## 1.3.1 (2026-10-19)

### Added
- `markAsPaid` field on `GET /expenses/paymentMethods`: an expense that gets this payment method is marked as paid.

## 1.3.0 (2026-10-19)

### Added
- `GET /expenses/paymentMethods`: list the payment methods of the expenses.
- `PATCH /expenses/status`: change the status of an expense.

## 1.2.3 (2026-10-13)

### Changed
- Invoice responses use the new `InvoiceResponseHeader` and `InvoiceResponseItem` schemas.
- `invoiceType` accepts only `F1`, `F2`, `R1` and `R5`.
- `irpf` accepts only `0`, `1`, `7` and `15`.
- The token documentation is clearer.

### Removed
- The `InvoiceResponse` schema.

## 1.2.2 (2026-10-13)

### Added
- `irpfPercent` and `irpfAmount` fields on expenses (IRPF withholding).

## 1.2.1 (2026-10-13)

### Changed
- New invoice statuses: `proforma`, `validating`, `rectified`, `rectifying` and `error`.
- `PATCH /invoice` accepts only the `paid` and `sent` statuses.
- The `PATCH /invoice` response includes `status` and `message`.

## 1.2.0 (2026-09-18)

### Added
- Invoice series: `GET /invoice/series` and `POST /invoice/series`.
- Billivo Expenses endpoints: `GET /expenses/categories`, `GET /expenses/export`, `GET /expenses/erp` and `POST /expenses/erp`.
- Invoice fields: `formatId` on the header, `base64pdf` on the invoice.
- Invoice item fields: `calificacionOperacion`, `operacionExenta`, `equivalenceSurcharge` and `equivalenceSurchargeTax`.
- `ValidationErrorsResponse` schema for validation errors.
- `pagination` and `message` on the invoice list response.

### Changed
- `taxType` accepts only `01`, `02`, `03` and `05`.

## 1.1.0 (2025-08-30 to 2025-12-10)

First public version.

### Added
- `POST /token` to get an access token.
- `GET /invoice`, `POST /invoice` and `PATCH /invoice`.
- Invoice header fields: `invoiceDate`, `correctedInvoiceNumber`, `previousInvoiceNumber`, `correctiveType`, `regimeKey`, `SWIFT`, `bankName` and `paymentTypeDescription`.
