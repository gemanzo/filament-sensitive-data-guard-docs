# Security model

Sensitive Data Guard is built to answer one question with evidence: **who saw which personal data, when and why.** That evidence is only worth something if the plugin is also the only way to see the data. This page explains what it guarantees, how you can check it yourself in two minutes, and where its protection ends.

## What is guaranteed

| Guarantee | How |
|---|---|
| A user without permission never receives the clear value | The decision is taken in PHP, before rendering. The browser gets `IT•• •••• … •••3 456`: not in the HTML, not in the Livewire snapshot, not in Livewire update responses. |
| Forms never leak the stored value | For users who may not see it, a sensitive input starts empty, with the masked value as placeholder. The component state, which Livewire sends to the browser, never contains the value. Leaving the field empty keeps the stored value. |
| Exports never leak it either | Export columns contain clear values only for users with standing permission. A temporary reveal does not count: it is for one value on screen, not for a file. |
| A reveal is recorded before the value is shown | The reveal, with its reason, is written to the database before the response that contains the value. If the request dies afterwards, the log entry is already there. |
| The log never contains the data it protects | Each entry stores who (user type, id and name at the time), which record (type and id), which field, the action, the reason and purpose, the page path without query string, IP address, browser and tenant. Never the value, and never the record title, which can be personal data too. |
| The log is append-only in the application | The model refuses updates and deletes. Old entries are removed only by `model:prune`, according to `retention.months`. |
| Tenants are isolated | In a panel with Filament tenancy, each entry stores its tenant, and the access log, the access history and the dashboard widget only show the current tenant's entries. |

## Check it yourself with the browser's developer tools

You need two users: one who sees masked values and one who may see them in clear (to know a clear value to search for).

1. **Find a clear value.** As the permitted user, open a customer and copy an IBAN or an e-mail address.
2. **Open the page as the other user.** Sign in, in a private window, as a user who sees masked values, and open the same customer list.
3. **Search the HTML.** Open the developer tools, *Elements* tab, press Ctrl+F (Cmd+F) and paste the clear value. No match. Search for `wire:snapshot`: that attribute holds the state Livewire sends to the browser. The value is not in it either.
4. **Search the network traffic.** *Network* tab, filter on `livewire`. Sort the table, change page, open the reveal dialog and cancel it. Open each `livewire/update` response and search for the value. No match.
5. **Check the edit form.** Open the customer's edit page. The sensitive input is empty and shows the masked value as a placeholder. Search `wire:snapshot` again: the field's state is `null`.
6. **Reveal it.** Click the masked value, choose a reason and add details. The value appears, for you only and for the configured minutes. As the permitted user, open the access log: the reveal is already there, with your reason.
7. **Export.** Still as the user without permission, export the customers. The file contains masked values.

## Why masking in the browser is not enough

Some tools hide values with CSS (a blur, a `•••` overlay) or JavaScript and show them on click. That protects the screen, not the data:

- the clear value is in the HTML and in the component state, so anyone can read it with the developer tools, a browser extension, a screen reader or a saved page;
- a "show" that only happens in the browser can only be logged if the browser reports the click, and a user can skip that report;
- a form that is filled with the real value sends it to the browser, whatever it displays.

Sensitive Data Guard does the opposite: the browser of a user without permission never receives the value, and every way to see it in clear goes through the server, where it is recorded.

## How the test suite holds the plugin to this

- Every leak test searches the Livewire payload as well as the HTML. Livewire's own `assertDontSee()` strips the component snapshot before searching, so a value leaking into the payload would go unnoticed; the plugin's tests pass `stripInitialData: false`.
- The leak tests have been checked by breaking the code on purpose (for example, filling the form with the real value): they fail.
- The suite runs on every supported combination of PHP 8.2–8.5, Laravel 11–13 and Filament 4 and 5, including the lowest allowed versions, and on MySQL and PostgreSQL as well as SQLite.

## Where the protection ends

The plugin protects what Filament renders through `->sensitive()`. It does not protect:

- **your own code that prints the raw attribute**: custom Blade views, `->description(fn ($record) => $record->iban)`, API resources, notifications, e-mails, application logs. Use `SensitiveDataGuard::maskAs($value, 'iban')` there;
- **users with standing permission**: they see values in clear, and each view is logged (once per user, record and field every 15 minutes by default);
- **the database itself**: masking is about who sees data in your panel. It does not encrypt data at rest. Combine it with Laravel's `encrypted` casts and restricted database access;
- **search**: a searchable column still finds a record when someone types the full value. Make sensitive columns non-searchable for users without permission if that matters to you;
- **the log against database administrators**: append-only is enforced by the application. For tamper evidence, forward the `SensitiveDataAccessed` event to your SIEM or to write-once storage;
- **queued jobs in multi-tenant apps**: exports run without a current tenant. Tell the plugin how to find a record's tenant with `->resolveTenantUsing()`, otherwise those entries have no tenant and only appear outside tenant panels.

Reveals and view de-duplication live in the cache. With more than one application server, use a shared cache store (`cache_store` in the config).
