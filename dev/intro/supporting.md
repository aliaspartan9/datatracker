# Supporting Applications

## Stats

The `stats` app is an older, lightly-used analytics layer over the datatracker data.
It provides statistics similar to the analyses historically compiled by individual
contributors. It has known limitations and receives relatively little active development.

The app contains four models:

| Model | Purpose |
|-------|---------|
| `MeetingRegistration` | Legacy registration data imported from the Secretariat's system |
| `AffiliationAlias` | Canonical form for organisation name variants |
| `AffiliationIgnoredEnding` | Regex patterns for corporate suffixes to strip (LLC, Inc., …) |
| `CountryAlias` | Maps country name variants to `CountryName` |

```mermaid
erDiagram
    MeetingRegistration {
        int id PK
        int meeting_id FK
        int person_id FK
        string first_name
        string last_name
        string affiliation
        string country_code
        string reg_type
        string ticket_type
        bool attended
        bool checkedin
    }
    AffiliationAlias {
        int id PK
        string alias UK
        string name
    }
    AffiliationIgnoredEnding {
        int id PK
        string ending
    }
    CountryAlias {
        int id PK
        string alias
        string country_id FK
    }
```

`MeetingRegistration` provides affiliation, country code, and name variations for a person
as recorded at meeting registration time — details that are not on the `Person` record
itself. The `AffiliationAlias` and `AffiliationIgnoredEnding` tables are attempts to
normalise affiliation strings; they are sparsely populated:

```python
from ietf.stats.models import AffiliationAlias, AffiliationIgnoredEnding

AffiliationAlias.objects.all()
# [<AffiliationAlias: cisco -> Cisco Systems>, ...]

AffiliationIgnoredEnding.objects.all()
# [<AffiliationIgnoredEnding: LLC\.?>, <AffiliationIgnoredEnding: Ltd\.?>, ...]
```

> **Note:** `stats.MeetingRegistration` is a legacy table for data imported before the
> `meeting.Registration` model existed. New registrations are stored in
> `meeting.Registration` (see [meeting.md](meeting.md)).

---

## IPR

The `ipr` app tracks Intellectual Property Rights disclosures submitted to the IETF.

`IprDisclosureBase` is the abstract base (implemented with multi-table inheritance). The
four concrete subtypes are:

| Subclass | Use case |
|----------|---------|
| `HolderIprDisclosure` | Patent holder disclosing their own patent |
| `ThirdPartyIprDisclosure` | Third-party disclosure of someone else's patent |
| `NonDocSpecificIprDisclosure` | Patent disclosure not tied to a specific document |
| `GenericIprDisclosure` | General IPR statement without patent specifics |

```mermaid
erDiagram
    IprDisclosureBase {
        int id PK
        string state_id FK
        string title
        string submitter_name
        string submitter_email
        bool compliant
        string notes
        datetime time
        bool by_default
    }
    HolderIprDisclosure {
        int iprdisclosurebase_ptr_id PK
        string holder_legal_name
        string patent_number
        string patent_inventor
        string patent_title
        string patent_date
        string patent_url
        string licensing_id FK
    }
    ThirdPartyIprDisclosure {
        int iprdisclosurebase_ptr_id PK
        string holder_legal_name
        string patent_number
        string patent_inventor
        string patent_title
    }
    NonDocSpecificIprDisclosure {
        int iprdisclosurebase_ptr_id PK
        string holder_legal_name
        string patent_number
        string patent_title
    }
    GenericIprDisclosure {
        int iprdisclosurebase_ptr_id PK
        string holder_legal_name
        string statement
    }
    IprDocRel {
        int id PK
        int disclosure_id FK
        int document_id FK
        string revisions
        string sections
    }
    RelatedIpr {
        int id PK
        int source_id FK
        int target_id FK
        string relationship
    }
    IprEvent {
        int id PK
        int disclosure_id FK
        string type_id FK
        int by_id FK
        datetime time
        string desc
    }
    RemovedIprDisclosure {
        int id PK
        string title
        string submitted_date
        string removed_date
        string removal_reason
    }

    IprDisclosureBase ||--o| HolderIprDisclosure : "is-a"
    IprDisclosureBase ||--o| ThirdPartyIprDisclosure : "is-a"
    IprDisclosureBase ||--o| NonDocSpecificIprDisclosure : "is-a"
    IprDisclosureBase ||--o| GenericIprDisclosure : "is-a"
    IprDisclosureBase ||--o{ IprDocRel : "iprdocrel_set"
    IprDisclosureBase ||--o{ IprEvent : "iprevent_set"
    IprDisclosureBase ||--o{ RelatedIpr : "source"
    IprDisclosureBase ||--o{ RelatedIpr : "target"
```

`IprDocRel` links a disclosure to the `Document` records it covers, with optional
revision and section ranges. `IprEvent` provides the audit trail for each disclosure,
recording state changes, inbound and outbound messages, comments, and legacy migration
events. `RemovedIprDisclosure` stores a minimal record for disclosures that were removed.

---

## Liaisons

The `liaisons` app manages formal liaison statements exchanged between the IETF and
external standards organisations (ITU, ISO, 3GPP, etc.). The sending and receiving
organisations are `Group` records with `type="sdo"` or similar.

```mermaid
erDiagram
    LiaisonStatement {
        int id PK
        string title
        string from_contact
        string to_contacts
        string response_contacts
        string technical_contacts
        string action_holder_contacts
        string cc_contacts
        string purpose_id FK
        date deadline
        string other_identifiers
        string body
        string state_id FK
    }
    LiaisonStatementAttachment {
        int id PK
        int liaisonstatement_id FK
        int document_id FK
        bool removed
    }
    RelatedLiaisonStatement {
        int id PK
        int source_id FK
        int target_id FK
        string relationship_id FK
    }
    LiaisonStatementEvent {
        int id PK
        int statement_id FK
        datetime time
        string type_id FK
        int by_id FK
        string desc
    }

    LiaisonStatement ||--o{ LiaisonStatementAttachment : "attachments"
    LiaisonStatement ||--o{ LiaisonStatementEvent : "liaisonstatementevent_set"
    LiaisonStatement ||--o{ RelatedLiaisonStatement : "source"
    LiaisonStatement ||--o{ RelatedLiaisonStatement : "target"
```

`LiaisonStatement` has M2M fields for `from_groups` and `to_groups` (both FK to
`Group`), and a M2M `tags` field (FK to `LiaisonStatementTagName`, values:
`action-required`, `action-taken`). Attached documents go through
`LiaisonStatementAttachment`, a through model that also tracks whether the attachment
has been removed.

`LiaisonStatementPurposeName` values: `for-action`, `for-comment`, `for-info`,
`in-response`, `other`.

---

## Nomcom

The `nomcom` app supports the IETF Nominations Committee process. Each year's NomCom is
represented by a `NomCom` record, which is associated with a `Group` record of
`type="nomcom"`.

```mermaid
erDiagram
    NomCom {
        int id PK
        int group_id FK
        string public_key
        bool is_accepting_nominations
        bool is_accepting_feedback
        bool send_questionnaire
        bool reminder_interval
        bool populate_temp_tables
    }
    Position {
        int id PK
        int nomcom_id FK
        string name
        bool is_open
        string description
        string requirements
        string questionnaire
    }
    Topic {
        int id PK
        int nomcom_id FK
        string subject
        string description
        bool accepting_feedback
        string audience_id FK
    }
    Nominee {
        int id PK
        int nomcom_id FK
        int email_id FK
        int person_id FK
    }
    NomineePosition {
        int id PK
        int position_id FK
        int nominee_id FK
        string state_id FK
    }
    Nomination {
        int id PK
        int position_id FK
        int nominee_id FK
        int nominator_email_id FK
        string comments
        bool share_nominator
    }
    Feedback {
        int id PK
        int nomcom_id FK
        int author_id FK
        string type_id FK
        string subject
        string comments
        datetime time
        bool person_seen
    }
    FeedbackLastSeen {
        int id PK
        int reviewer_id FK
        int nominee_id FK
        datetime time
    }
    Volunteer {
        int id PK
        int nomcom_id FK
        int person_id FK
        string origin
    }
    ReminderDates {
        int id PK
        int nomcom_id FK
        date nomination_reminder_date
        date accept_reminder_date
    }

    NomCom ||--o{ Position : "position_set"
    NomCom ||--o{ Topic : "topic_set"
    NomCom ||--o{ Nominee : "nominee_set"
    NomCom ||--o{ Feedback : "feedback_set"
    NomCom ||--o{ Volunteer : "volunteer_set"
    NomCom ||--o| ReminderDates : "reminderdates"
    Position ||--o{ NomineePosition : "nomineeposition_set"
    Nominee ||--o{ NomineePosition : "nomineeposition_set"
    Position ||--o{ Nomination : "nomination_set"
    Nominee ||--o{ Nomination : "nomination_set"
    Nominee ||--o{ FeedbackLastSeen : "feedbacklastseen_set"
```

`Feedback` content (nominations, questionnaire responses, and comments) is encrypted
with the NomCom's GPG public key stored in `NomCom.public_key`. This means feedback
text stored in the database is ciphertext and can only be decrypted by NomCom members
who hold the corresponding private key.

`NomineePositionStateName` values: `accepted`, `declined`, `none`.

`FeedbackTypeName` values: `questionnaires`, `nominations`, `comments`. Each type has a
`legend` field used in the UI.

`Volunteer` records track people who have volunteered to serve on the NomCom (as
distinguished from being nominated for a position). The `origin` field records how the
volunteer record was created.
