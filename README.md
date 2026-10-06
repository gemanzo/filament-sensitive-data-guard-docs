# Sensitive Data Guard for Filament

**Mask personal data in your Filament panels, require a reason to reveal it, and keep an audit trail of who saw what.**

Your support team needs the customer list. They don't need every IBAN, phone number and tax code in clear text on every screen. Your auditor, your DPO, or the regulator will eventually ask: *who looked at this customer's data, and why?*

Sensitive Data Guard answers both with one method:

```php
TextColumn::make('iban')->sensitive('iban'),
```

![Masked customer table, the reveal dialog asking for a reason, and an alert to the DPO](https://raw.githubusercontent.com/gemanzo/filament-sensitive-data-guard-docs/main/images/banner.jpg)

## What it does

- **Masks on the server.** Users without permission get `IT•• •••• •••• •••• •••• •••3 456`. The clear value never reaches their browser: not in the HTML, not in the Livewire payload.
- **Break the glass, with a reason.** Masked values have a *Show* action. The user picks a reason, adds details (ticket number, request reference) and sees the value for a few minutes. The reveal is logged immediately.
- **Logs every access, never the value.** Views by permitted users, reveals and exports are written to an append-only access log, with user, record, field, reason, page, IP and time.
- **Flags unusual access.** Too many records revealed in an hour, the same record revealed again and again, or bulk browsing of unmasked data raise an alert to your DPO.
- **Answers "who saw this?" in one click.** An access history action on any record, and a read-only access log with filters and CSV export for audits.
- **Covers forms and exports too.** Edit forms become write-only for users who can't see a value; Filament exports contain clear values only for users with standing permission.

Built-in maskers: `iban`, `card`, `email`, `phone`, `tax_id`, `name`, `date_of_birth`, `partial`, `full` — or your own.

Works with **Filament 4 (4.1.8+) and 5**, Laravel 11–13, PHP 8.2+. English and Italian translations included.

## Why

- **GDPR** asks for appropriate technical measures, including access control and logging of access to personal data (art. 32), and for being able to demonstrate compliance (art. 5(2)).
- **Supervisory authorities enforce it.** The Italian Garante, for instance, requires banks to track employee access to customer data, keep those logs for 24 months and alert on anomalous access; health-care providers have been fined for lacking access controls and anomaly alerts.
- **HIPAA** (audit controls) and **SOC 2** ask the same question: who accessed sensitive data, when, and was it legitimate?

This plugin is a technical building block, not legal advice. Check with your DPO or lawyer what applies to you.

## Installation

Add the private repository you received with your license, then:

```bash
composer require gennaromanzo/filament-sensitive-data-guard
php artisan sensitive-data-guard:install
```

The installer publishes the config file and the migration and offers to run it.

Register the plugin in your panel provider:

```php
use GennaroManzo\SensitiveDataGuard\SensitiveDataGuardPlugin;

public function panel(Panel $panel): Panel
{
    return $panel
        // ...
        ->plugin(SensitiveDataGuardPlugin::make());
}
```

Keep the log from growing forever (see `retention.months` in the config):

```php
// routes/console.php
use GennaroManzo\SensitiveDataGuard\Models\AccessLog;

Schedule::command('model:prune', ['--model' => [AccessLog::class]])->daily();
```

## Decide who sees what

There are three permissions. Define them as gates, or as closures on the plugin (closures win).

| Permission | Gate | Plugin method | Default |
|---|---|---|---|
| Always see values in clear (every view is logged) | `viewSensitiveData` | `->viewUnmaskedUsing()` | nobody |
| Reveal a value by giving a reason | `revealSensitiveData` | `->revealUsing()` | any signed-in user (configurable) |
| Open the access log and access history | `viewSensitiveDataAccessLog` | `->viewAccessLogUsing()` | your Filament policies |

```php
// AppServiceProvider::boot()
Gate::define('viewSensitiveData', fn (User $user, string $category, ?Model $record) =>
    $user->hasRole('dpo') || ($category === 'email' && $user->hasRole('sales'))
);

Gate::define('revealSensitiveData', fn (User $user) => $user->hasRole('support'));

Gate::define('viewSensitiveDataAccessLog', fn (User $user) => $user->hasRole('dpo'));
```

The `$category` is the masker name (`iban`, `email`, …) unless you set your own:

```php
TextColumn::make('diagnosis')->sensitive('full', category: 'health'),
```

Or on the plugin:

```php
SensitiveDataGuardPlugin::make()
    ->viewUnmaskedUsing(fn (User $user, string $category) => $user->can("see {$category}"))
    ->revealUsing(fn (User $user) => $user->hasRole(['support', 'billing']))
    ->viewAccessLogUsing(fn (User $user) => $user->hasRole('dpo'))
    ->alertRecipientsUsing(fn () => User::role('dpo')->get())
    ->revealDuration(minutes: 5);
```

## Usage

Call `->sensitive()` **last** in the chain: it wraps the formatting, tooltip and click action configured before it.

### Tables

```php
TextColumn::make('iban')->sensitive('iban'),
TextColumn::make('email')->sensitive('email'),
TextColumn::make('phone')->sensitive('phone', revealable: false), // masked, no "Show"
TextColumn::make('customer.tax_code')->sensitive('tax_id'),       // logged against the customer
```

Masked cells show a lock icon; clicking one opens the reveal dialog. After a reveal, the value appears in place for the configured number of minutes.

![Customer table with masked e-mail and IBAN columns](https://raw.githubusercontent.com/gemanzo/filament-sensitive-data-guard-docs/main/images/masked-table.jpg)

![Reveal dialog asking for a reason and details](https://raw.githubusercontent.com/gemanzo/filament-sensitive-data-guard-docs/main/images/reveal-dialog.jpg)

### Infolists

```php
TextEntry::make('iban')->sensitive('iban'),
```

The *Show* action appears next to the label.

![Customer page with masked entries and Show actions](https://raw.githubusercontent.com/gemanzo/filament-sensitive-data-guard-docs/main/images/infolist-masked.jpg)

### Forms

```php
TextInput::make('iban')->required()->sensitive('iban'),
```

For users who can't see the value, the input starts empty with the masked value as placeholder. Leaving it empty keeps the stored value; typing replaces it. `required()` is relaxed accordingly. A *Show* action next to the input reveals the value, with a reason.

![Edit form where sensitive fields are write-only](https://raw.githubusercontent.com/gemanzo/filament-sensitive-data-guard-docs/main/images/write-only-form.jpg)

### Exports

```php
ExportColumn::make('iban')->sensitive('iban'),
```

Clear values only for users with standing permission (a temporary reveal does not count: it is for one value on screen, not for a file). Each exported clear value is logged.

### Declare once, on the model

```php
use GennaroManzo\SensitiveDataGuard\Concerns\HasSensitiveAttributes;

class Customer extends Model
{
    use HasSensitiveAttributes;

    protected array $sensitive = [
        'iban' => 'iban',
        'email' => 'email',
        'tax_code' => 'tax_id',
    ];
}

TextColumn::make('iban')->sensitive(); // uses the "iban" masker
```

### Who saw this record?

```php
use GennaroManzo\SensitiveDataGuard\Filament\Actions\ViewAccessHistoryAction;

protected function getHeaderActions(): array
{
    return [ViewAccessHistoryAction::make()];
}
```

It works as a table record action too. Only users allowed to open the access log see it.

![Access history of a customer: views and reveals with reasons](https://raw.githubusercontent.com/gemanzo/filament-sensitive-data-guard-docs/main/images/access-history.jpg)

### Custom maskers

```php
SensitiveDataGuardPlugin::make()
    ->maskUsing('plate', fn (string $value) => substr($value, 0, 2) . '•••' . substr($value, -2));

TextColumn::make('license_plate')->sensitive('plate'),
```

A masker can also be a class implementing `GennaroManzo\SensitiveDataGuard\Masking\Masker`, or a closure passed directly: `->sensitive(fn (string $value) => '••• ' . substr($value, -3))`.

## The access log

The plugin adds a read-only **Sensitive data access** resource (group *Compliance*): filters by action, record type, user and date, a detail page per entry, and CSV download. Entries cannot be edited or deleted through the application; old entries are pruned by `model:prune`.

![Read-only access log with user, record, field and reason](https://raw.githubusercontent.com/gemanzo/filament-sensitive-data-guard-docs/main/images/access-log.jpg)

A dashboard widget shows reveals, unmasked views and exported values of the last 24 hours (counts only). Turn either off with `->accessLogResource(false)` or `->overviewWidget(false)`.

![Dashboard widget with reveals, unmasked views and exported values](https://raw.githubusercontent.com/gemanzo/filament-sensitive-data-guard-docs/main/images/dashboard-widget.jpg)

For audits and data subject requests:

```bash
php artisan sensitive-data-guard:report --from=2026-01-01 --to=2026-03-31 --output=storage/app/access-q1.csv
php artisan sensitive-data-guard:report --subject-type="App\Models\Customer" --subject-id=42
```

The CSV neutralises spreadsheet formulas typed into reasons.

What is stored per access: user (type, id and name at the time), record (type and id), field, category, action, reason, panel, page path (no query string), IP address and user agent (both optional), and context. **Never the value.** Record titles are not stored either: they can be personal data themselves.

## Alerts

After each write to the log, the plugin checks the user's activity in the last hour (all configurable):

| Rule | Default |
|---|---|
| Different records revealed | more than 10 |
| Different records viewed unmasked | more than 200 |
| Reveals of the same record | more than 3 |

When a rule is broken, `SuspiciousAccessDetected` is dispatched (once per user per cooldown period), a warning is written to your application log, and the users returned by `->alertRecipientsUsing()` receive a Filament database notification. Listen to the event for e-mail, Slack or your SIEM. Every access also dispatches `SensitiveDataAccessed`.

![Database notification to the DPO about unusual access](https://raw.githubusercontent.com/gemanzo/filament-sensitive-data-guard-docs/main/images/alert-notification.jpg)

The notification is a standard Filament database notification, so it needs what those always need:

```php
// Your panel provider
$panel->databaseNotifications();
```

```bash
php artisan make:notifications-table
php artisan migrate
php artisan queue:work   # Filament queues database notifications
```

Without a queue worker the alert waits in the `jobs` table and never reaches the bell icon.

## Configuration

```bash
php artisan vendor:publish --tag=sensitive-data-guard-config
```

Highlights: mask character, reveal duration, reason list, whether details are required, view de-duplication window, IP / user agent storage, retention, alert thresholds, cache store. Every option is documented in the file.

**More than one app server?** Reveals and de-duplication live in the cache: set `cache_store` to a shared store (Redis, Memcached, database).

## Good to know

- The plugin protects what Filament renders. Your own code that prints the raw attribute (custom views, `->description(fn ($record) => $record->iban)`, API resources, notifications) is not masked: use `SensitiveDataGuard::maskAs($customer->iban, 'iban')` there.
- Searching a sensitive column with a full value still finds the record. If that is a concern, make the column non-searchable for users without permission.
- `->sensitive()` replaces the column's click action. A column with `->url()` keeps its URL and offers no in-place reveal.
- Access logs are written at the end of the request (one bulk insert) or of the queued job. Reveals are written immediately. Set `log.strategy` to `immediate` to write every access right away.
- Masking is about who *sees* data in your panel. It does not encrypt data at rest: combine it with Laravel's `encrypted` casts for that.

## Testing

```bash
composer test
```

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## Support

Open an issue in the repository you received with your license, or e-mail the address on your receipt.

## License

Commercial. See [LICENSE.md](LICENSE.md).
