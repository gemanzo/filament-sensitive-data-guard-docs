# Changelog

All notable changes to `filament-sensitive-data-guard` are documented here.

## 1.0.0 — unreleased

- `->sensitive()` for `TextColumn`, `TextEntry`, `TextInput` and `ExportColumn`
- Server-side masking: clear values never reach the browser of users without permission
- Reveal with a reason, temporary in-place access, logged immediately
- Append-only access log with views, reveals and exports — never the values
- Relationship fields logged against the record that owns the data
- Write-only edit forms for users who may not see a value
- Built-in maskers: iban, card, email, phone, tax_id, name, date_of_birth, partial, full; custom maskers
- Permissions via gates or plugin closures, per category and per field
- Anomaly alerts (bulk reveals, bulk unmasked views, repeated reveals) with cooldown and database notifications
- Read-only access log resource with filters and CSV download, record access history action, dashboard widget
- `sensitive-data-guard:report` command for audits and data subject requests
- Retention through `model:prune`
- English and Italian translations
- Filament 4 (4.1.8+) and 5, Laravel 11–13, PHP 8.2+
