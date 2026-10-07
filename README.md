# Sensitive Data Guard for Filament

**Prove who saw which personal data, when and why.**

Your auditor, your DPO, a regulator or the customer themselves will eventually ask: *who looked at this person's data, and for what reason?* Sensitive Data Guard gives your Filament panels the answer, with evidence: an append-only access log of every view, reveal and export of sensitive data, with a purpose for each one, a report for data subject requests and alerts on unusual access.

To make that log complete, the data is masked on the server for everyone without permission, and seeing it in clear takes a reason. One method does it:

```php
TextColumn::make('iban')->sensitive('iban'),
```

![Masked customer table, the reveal dialog asking for a reason, and an alert to the DPO](https://raw.githubusercontent.com/gemanzo/filament-sensitive-data-guard-docs/main/images/banner.jpg)

## What it does

- **Logs every access, never the value.** Views by permitted users, reveals and exports are written to an append-only access log: user, record, field, action, reason or purpose, page, IP, time and tenant. Never the value, never the record title.
- **Records why.** A reveal requires a reason (support ticket, request from the data subject, fraud check…) plus details. Ordinary views and exports get a purpose too, set per role.
- **Answers data subject requests.** One command produces, for one person, the dates, data and purposes of every access, with staff names withheld as the EU Court of Justice allows (case C-579/21).
- **Flags unusual access.** Too many records revealed in an hour, the same record revealed again and again, or bulk browsing of unmasked data raise an alert to your DPO.
- **Answers "who saw this?" in one click.** An access history action on any record, and a read-only access log with filters and CSV export for audits.
- **Masks on the server.** Users without permission get `IT•• •••• •••• •••• •••• •••3 456`. The clear value never reaches their browser: not in the HTML, not in the Livewire payload, not in form state, not in exports. A log that the browser could bypass would prove nothing. See the [security model](docs/security-model.md) to check it yourself with the developer tools.
- **Multi-tenant ready.** In panels with Filament tenancy, each tenant only sees its own access log.

Built-in maskers: `iban`, `card`, `email`, `phone`, `tax_id`, `name`, `date_of_birth`, `partial`, `full` — or your own.

Works with **Filament 4 (4.1.8+) and 5**, Laravel 11–13, PHP 8.2+. English and Italian translations included.

## Why

- **GDPR** asks you to be able to demonstrate compliance (art. 5(2)), to protect personal data by default (art. 25) and to make sure staff only process it on instructions (art. 32). Data subjects may ask when and why their data was consulted (art. 15, CJEU C-579/21).
- **Supervisory authorities enforce it.** The Italian Garante, for instance, requires banks to track every employee access to customer data, simple consultations included, keep those logs for 24 months and alert on anomalous access; health-care providers have been fined for lacking access controls and anomaly alerts.
- **HIPAA** (audit controls, activity review), **SOC 2** (CC6, CC7) and **ISO/IEC 27001** (A.8.11 data masking, A.8.15 logging) ask the same question: who accessed sensitive data, when, and was it legitimate?

The [compliance mapping](docs/compliance.md) lists, rule by rule, what the plugin provides and what remains up to you. This plugin is a technical building block, not legal advice. Check with your DPO or lawyer what applies to you.

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

## Record why: purposes

Every access has a purpose. A reveal records the reason the user chose (and the details typed in the dialog). For ordinary views and exports by users with standing permission, set the purpose per role:

```php
SensitiveDataGuardPlugin::make()
    ->defaultPurposeUsing(fn (User $user) => match (true) {
        $user->hasRole('dpo') => 'legal_obligation',   // a reveal reason code: translated in reports
        $user->hasRole('billing') => 'Invoicing and payments',
        default => 'Customer support',
    });
```

The closure also receives `$category`, `$record`, `$field` and `$action` (`AccessAction::View` or `AccessAction::Export`) by name. Without a closure, `log.default_purpose` in the config applies to everyone.

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

For audits, the full report of a period or of one record:

```bash
php artisan sensitive-data-guard:report --from=2026-01-01 --to=2026-03-31 --output=storage/app/access-q1.csv
php artisan sensitive-data-guard:report --subject-type="App\Models\Customer" --subject-id=42
```

The CSV neutralises spreadsheet formulas typed into reasons.

What is stored per access: user (type, id and name at the time), record (type and id), field, category, action, reason, purpose, panel, tenant, page path (no query string), IP address and user agent (both optional), and context. **Never the value.** Record titles are not stored either: they can be personal data themselves.

## Data subject requests

When a person asks who consulted their data (GDPR art. 15), produce the report for them:

```bash
php artisan sensitive-data-guard:report --for-data-subject \
    --subject-type="App\Models\Customer" --subject-id=42 --locale=it \
    --output=storage/app/access-customer-42.csv
```

| Date | Personal data | Access | Purpose | Accessed by |
|---|---|---|---|---|
| 2026-03-02 10:15 UTC | IBAN | Revealed | Support ticket | Authorised staff member |
| 2026-03-05 16:40 UTC | E-mail address | Viewed | Legal or regulatory obligation | Authorised staff member |

The EU Court of Justice (case C-579/21, *Pankki S*, 2023) held that a data subject is entitled to the dates and purposes of consultations of their data, but not in principle to the identity of the employees who acted on the controller's instructions. So the report withholds staff names, and leaves out IP addresses, pages and the free-text details of reveals, which may concern other people. If you decide that an employee's identity is essential in a specific case, the full report above has it.

Accesses to related records are included: a field shown through a relationship (`customer.iban` on an order) is logged against the customer who owns it.

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

## Multi-tenant panels

In a panel with Filament tenancy (`->tenant(Team::class)`), each access records the current tenant, and the access log, the access history and the dashboard widget only show the current tenant's entries: one customer's DPO never sees another customer's log. Panels without tenancy, such as a super-admin panel, see every entry.

Queued exports run without a current tenant. Tell the plugin how to find a record's tenant:

```php
SensitiveDataGuardPlugin::make()
    ->resolveTenantUsing(fn (Model $record) => $record->team);
```

## Configuration

```bash
php artisan vendor:publish --tag=sensitive-data-guard-config
```

Highlights: mask character, reveal duration, reason list, whether details are required, default purpose, view de-duplication window, IP / user agent storage, retention, alert thresholds, cache store. Every option is documented in the file.

**More than one app server?** Reveals and de-duplication live in the cache: set `cache_store` to a shared store (Redis, Memcached, database).

## Good to know

- The plugin protects what Filament renders. Your own code that prints the raw attribute (custom views, `->description(fn ($record) => $record->iban)`, API resources, notifications) is not masked: use `SensitiveDataGuard::maskAs($customer->iban, 'iban')` there.
- Searching a sensitive column with a full value still finds the record. If that is a concern, make the column non-searchable for users without permission.
- `->sensitive()` replaces the column's click action. A column with `->url()` keeps its URL and offers no in-place reveal.
- Access logs are written at the end of the request (one bulk insert) or of the queued job. Reveals are written immediately. Set `log.strategy` to `immediate` to write every access right away.
- Masking is about who *sees* data in your panel. It does not encrypt data at rest: combine it with Laravel's `encrypted` casts for that.
- The [security model](docs/security-model.md) lists exactly what is guaranteed and where the protection ends.

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
