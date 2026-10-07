# Compliance mapping

Sensitive Data Guard is a technical building block. It gives you the controls and the evidence that several rules ask for, but compliance also depends on your processes, your contracts and how you configure the plugin. **This page is not legal advice:** check with your DPO, auditor or lawyer what applies to you.

Each row says what the plugin provides and what remains your responsibility.

## GDPR

| Requirement | What the plugin provides | What you still need |
|---|---|---|
| **Art. 5(2) — accountability:** be able to demonstrate compliance | Append-only access log: who saw which personal data, when, how (view, reveal, export) and why. Access history per record, read-only log with filters, CSV report for any period. | Decide the retention period, review the log, describe the processing in your records (art. 30). |
| **Art. 25 — data protection by default** | Values are masked for everyone who has no standing permission. The log stores no values and no record titles. | Decide which roles need values in clear, and keep that list short. |
| **Art. 32(1)(b) — confidentiality;** **art. 32(4)** — staff process personal data only on instructions | Masking on the server, reveal only with a reason, per-category permissions, alerts on unusual access (bulk reveals, bulk unmasked views, the same record revealed again and again). | Instructions and training for staff, encryption at rest, access control to the database and backups. |
| **Art. 15 — right of access**, as interpreted by the CJEU in **case C-579/21** (*Pankki S*, 22 June 2023): the data subject is entitled to the **dates and purposes** of consultations of their data, not in principle to the identity of the employees who acted on the controller's instructions — unless that identity is essential for them to exercise their rights, taking the employees' rights into account | `php artisan sensitive-data-guard:report --for-data-subject`: date, data, type of access and purpose for one person, staff names withheld, also in Italian (`--locale=it`). IP addresses, pages and the free-text details of reveals are left out, as they may concern other people. | Record a purpose for ordinary views too (`->defaultPurposeUsing()`, see the README). Decide case by case whether an employee's identity is essential: the full report (without `--for-data-subject`) is available to the DPO. |

## Italian Garante — banking sector (provvedimento of 12 May 2011)

| Requirement | What the plugin provides | What you still need |
|---|---|---|
| Trace every access to customer data, **including simple consultation**: employee, date and time, workstation, customer and relationship consulted | Views, reveals and exports through Filament are logged with user id and name, date and time, IP address and browser, record and field. | Access outside Filament (core banking, other applications) needs its own tracing. If your workstation code is not the IP address, store it through the `SensitiveDataAccessed` event. |
| Keep the logs for **at least 24 months** | `retention.months` defaults to 24; older entries are removed by `model:prune`. | Schedule `model:prune`, and do not lower the retention. |
| **Alerts** on anomalous or risky behaviour, such as mass consultations or repeated access to the same customer | Rules for bulk reveals, bulk unmasked views and repeated reveals of the same record, with configurable thresholds; event plus Filament notification to the DPO. | Tune the thresholds, route the alert where someone acts on it, document the follow-up. |
| A documented **internal audit at least once a year**, by staff without access to customer data | The access log, filters and CSV report are the material for that audit. | The audit itself. |

## HIPAA Security Rule (45 CFR Part 164, Subpart C)

| Requirement | What the plugin provides | What you still need |
|---|---|---|
| **§164.312(b) — audit controls:** record and examine activity in systems containing ePHI | Every view, reveal and export of a sensitive field in Filament is recorded. | Audit controls for the rest of your systems. |
| **§164.308(a)(1)(ii)(D) — information system activity review** | Read-only access log with filters, CSV report, record access history, dashboard widget, anomaly alerts. | A documented, regular review. |
| **§164.312(a)(1) — access control;** minimum necessary (§164.502(b)) | Masking by default, permissions per category of data, reveal only with a reason. | Role definitions, user provisioning and termination. |
| Documentation retention (§164.316(b)(2)) is six years; many organisations apply it to audit logs | Retention is configurable. | Set `retention.months` to 72 if you follow that practice. |

## SOC 2 (AICPA Trust Services Criteria)

| Criterion | What the plugin provides | What you still need |
|---|---|---|
| **CC6.1** — logical access security over protected information | Clear values only for users allowed by your gates or plugin rules; everyone else gets masked values from the server. | Authentication, MFA, access reviews. |
| **CC6.3** — role-based access, least privilege | Permissions per category of data (`iban`, `health`…), per field when needed. | Mapping roles to permissions, periodic review. |
| **CC7.2** — monitoring for anomalies | Anomaly alerts, `SensitiveDataAccessed` event for your SIEM, dashboard widget. | Incident response for the alerts. |
| Evidence for the auditor | CSV report of all accesses in the audit period. | — |

## ISO/IEC 27001:2022 (Annex A)

| Control | What the plugin provides | What you still need |
|---|---|---|
| **A.8.11 — data masking** | Built-in maskers (`iban`, `card`, `email`, `phone`, `tax_id`, `name`, `date_of_birth`, `partial`, `full`) and custom ones, applied on the server. | A masking policy: which data, for whom. |
| **A.8.15 — logging** | Access log of sensitive data, append-only in the application. | Protecting logs against tampering at database level, clock synchronisation. |
| **A.8.16 — monitoring activities** | Anomaly alerts. | The monitoring process. |
| **A.5.15 — access control** | Permissions for clear values, reveals and the access log. | The access control policy. |

## Sources

- GDPR: [Regulation (EU) 2016/679](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
- CJEU, case C-579/21: [press release No 107/23](https://curia.europa.eu/jcms/upload/docs/application/pdf/2023-06/cp230107en.pdf)
- Garante per la protezione dei dati personali, provvedimento of 12 May 2011: [docweb 1813953](https://www.garanteprivacy.it/garante/doc.jsp?ID=1813953)
- HIPAA Security Rule: [45 CFR Part 164, Subpart C](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-C)
