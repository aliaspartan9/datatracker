# Datatracker Database Overview

This set of documents describes the Django models that make up the IETF Datatracker database.
It is intended as an orientation for developers new to the codebase and for data researchers
who want to query the database directly.

The datatracker source lives at <https://github.com/ietf-tools/datatracker>. The REST API
is available at <https://datatracker.ietf.org/api/v1/>.

## Applications

The datatracker is a Django project. Each logical area of functionality lives in a Django app
under `ietf/`. The apps covered here are:

| App | Purpose |
|-----|---------|
| [person](person.md) | People, email addresses, and API keys |
| [doc](document.md) | Documents — drafts, RFCs, charters, meeting materials, and more |
| [group](group.md) | Working groups, areas, directorates, teams, and their roles |
| [meeting](meeting.md) | IETF meetings, sessions, scheduling, and proceedings |
| [review](review.md) | Document review teams, requests, and assignments |
| [supporting](supporting.md) | Stats, IPR disclosures, liaison statements, and NomCom |

Supporting apps whose models are not documented in depth here include:

- `community` — personal and group document watch lists
- `dbtemplate` — database-stored Django/RST templates
- `iesg` — telechat dates and agenda items
- `mailtrigger` — rules and recipient sets for automated email
- `message` — outbound email messages and send queues
- `submit` — Internet-Draft submission workflow
- `name` — enumeration tables (slug/name pairs) used as foreign keys throughout

## Name models

Almost every foreign key that looks like a "type" or "state" field points to a row in a
small enumeration table that subclasses `NameModel` from `ietf/name/models.py`. These
tables have the shape:

```
slug (PK)  |  name  |  desc  |  used  |  order
```

Examples: `DocTypeName`, `GroupTypeName`, `RoleName`, `StreamName`, `StateType`, `ReviewResultName`.
When you see a field like `type_id` or `state_id` on a model, it is almost certainly a FK to
one of these tables.

## Querying with the Django shell

A local development environment gives you a Django shell:

```shell
$ ./ietf/manage.py shell
```

Or via the REST API:

```shell
$ curl "https://datatracker.ietf.org/api/v1/doc/document/?format=json&limit=1" | jq
```

## Key data-modelling conventions

- **Primary keys** are auto-increment integers unless otherwise noted. `NameModel`
  subclasses use the `slug` as PK.
- **Historical records** — several core models (Person, Email, ReviewRequest, etc.) attach
  `simple_history.models.HistoricalRecords` for full audit trails.
- **DocEvent audit trail** — document history is primarily tracked through append-only
  `DocEvent` rows; see [document.md](document.md).
- **Soft references by name** — `Document.name` is a unique, immutable string key used in
  URLs and inter-model references (e.g. `RelatedDocument.target` is a FK to `Document`).
  There is no longer a `DocAlias` indirection layer; see [document.md](document.md).
