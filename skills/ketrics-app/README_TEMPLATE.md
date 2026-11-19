# <App name>

<!--
  Template for the app's README.md (project root). Required by the ketrics-app skill, section
  "README (required)". Describe the app as it works today, in business language. Delete these
  comments, and any section that does not apply, but keep the eight headings in this order.
  Written for people, business users included: no instructions, notes or prompts addressed to an
  AI, assistant or agent. No handler names, SQL, environment variable names or code. No secrets or
  credentials, no URLs, IP addresses, hostnames, file paths or account, bucket or resource
  identifiers, no real people, customers, amounts, prices or contract terms (use made-up examples).
  Keep it under 100 KB. Update it in the same PR as the flow, rule, role or integration it describes.
-->

**Version:** 1.0.0 · **Last updated:** YYYY-MM-DD

## 1. Purpose

<!-- The business problem this app solves, and who uses it (for example: the accounts team, monthly). -->

## 2. Roles and permissions

<!-- One line per role, in the role's business name: what the people in that role can see and do.
     Example: "Approver: reviews submitted requests and approves or rejects them." -->

| Role | Can do |
| --- | --- |
| <Role name> | <What they can see and do> |

## 3. Screens and flows

<!-- The main screens, then each flow as numbered steps a user follows, from start to finish. -->

### <Flow name>

1. <Step>
2. <Step>

## 4. Data entities

<!-- What the app keeps, in plain words: the things users talk about, and what each one holds.
     Example: "Request: a purchase need raised by one person, with a description, an amount and a status." -->

- **<Entity>:** <what it is and what it holds>

## 5. Business rules

<!-- Validations, calculations, statuses and approvals, each stated as a rule a user would recognise.
     Example: "A request above 5,000 needs a second approver before it can be paid." -->

- <Rule>

## 6. Integrations

<!-- What the app receives from and sends to other systems: the system, the direction, the format
     and how often. Name the business data, not the technical endpoint (no URLs or identifiers). -->

| System | Direction | Data | Format and frequency |
| --- | --- | --- | --- |
| <System> | In / Out | <Business data> | <CSV, Excel, API; daily, on demand> |

## 7. Known limits and out of scope

<!-- What the app does not do, and the limits that apply (file sizes, number of rows, supported
     browsers, languages). -->

- **Out of scope:** <what it does not do>
- **Limits:** <limit and its value>

## 8. Version and changelog

<!-- The current version at the top, then one dated line per change, newest first. -->

| Version | Date | Change |
| --- | --- | --- |
| 1.0.0 | YYYY-MM-DD | First version |
