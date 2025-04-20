# Topcoder Challenge API – Data Migration Tool

This tool migrates challenge-related data from JSON/NDJSON files (exported from DynamoDB or ElasticSearch) into a **PostgreSQL** database using **Prisma ORM**. It supports multiple models of the **Challenge API**, including complex nested structures like prize sets, discussions, and event phases.

---

## ✅ Supported Models

The following models are migrated (if valid data is available):

- `Challenge`
- `ChallengeMetadata`
- `ChallengePrizeSet`
- `Prize`
- `ChallengeWinner`
- `ChallengeTerm`
- `ChallengeSkill`
- `ChallengeBilling`
- `ChallengeLegacy`
- `ChallengeEvent`
- `ChallengeDiscussion`
- `ChallengeDiscussionOption`
- `ChallengeConstraint`
- `ChallengePhase`
- `ChallengePhaseConstraint`
- `ChallengeTrack`
- `ChallengeType`
- `Phase`
- `TimelineTemplate`
- `ChallengeTimelineTemplate`

### ❌ Skipped Models (due to missing references)

- `AuditLog`
- `Attachment`

Both were skipped due to their `challengeId` references not being found in the provided challenge dataset (`challenge-api.challenge.json`).

---

## 📦 Technologies Used

- **Node.js**
- **Prisma ORM**
- **PostgreSQL 16.3** (via Docker)
- **Docker Compose**
- **stream-json** and **readline**
- **Jest** (for unit testing)

---

## ⚙️ Environment Setup

Create a `.env` file in the root directory:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/challengedb"
CREATED_BY="challenge-api-db-migration"
```

> You can override `CREATED_BY` at runtime if needed:
```bash
CREATED_BY="your-name" node src/index.js challenges ./data/sample.ndjson
```

---

## 🚀 Running the Migration

1. Install dependencies:

```bash
npm install
```

2. Start PostgreSQL with Docker:

```bash
docker-compose up -d
```

3. Push Prisma schema:

```bash
npx prisma db push
```

4. Run individual migration steps:

```bash
node src/index.js challenge-types
node src/index.js challenges ./data/challenges_sample.ndjson
```
Avaliable "steps" down below.

5. (Optional) Run all in sequence:

```bash
node src/index.js migrate-all
```

---

## 🧩 Migration Strategy

- Most models use **streaming migration** (`readline` or `stream-json`) to support large files.
- Submodels like `ChallengeMetadata`, `ChallengePrizeSet`, `Prize`, and `ChallengeWinner` are migrated **within the main `Challenge` migration** step.

---

## 🧩 Available Migration Steps

| Step                           | Description                                                                 | Default File Path                                       |
|--------------------------------|-----------------------------------------------------------------------------|----------------------------------------------------------|
| `challenge-types`             | Imports `ChallengeType` definitions                                        | `./data/ChallengeType_dynamo_data.json`                 |
| `challenge-tracks`            | Imports `ChallengeTrack` definitions                                       | `./data/ChallengeTrack_dynamo_data.json`               |
| `phases`                      | Imports global `Phase` definitions                                         | `./data/Phase_dynamo_data.json`                         |
| `timeline-templates`          | Imports `TimelineTemplate` base structures and associated phases           | `./data/TimelineTemplate_dynamo_data.json`              |
| `challenge-timeline-templates`| Links challenge types to timeline templates                                | `./data/ChallengeTimelineTemplate_dynamo_data.json`     |
| `attachments`                 | Implemented, but no data migrated due to missing `challengeId` references  | `./data/Attachment_dynamo_data.json`                    |
| `audit-logs`                  | Implemented, but no data migrated due to missing `challengeId` references  | `./data/AuditLog_dynamo_data.json`                      |
| `challenges`                  | Full challenge migration with all submodels (skills, winners, etc.)        | `./data/challenge-api.challenge.json`                   |


All with auto strategy implemented uses `stream-json` (batch) for files larger than 3MB, and `loadJSON` (simple) otherwise.

  > ⚙️ **Why Auto Strategy?**
>
> For models that involve large datasets (`member-profiles`, `member-stats`, and `resources`), the tool implements an **automatic selection strategy** based on file size:
> - If the input file is **larger than 3 MB**, the migration runs in **batch mode using streaming (e.g., `stream-json` or `readline`)** to reduce memory usage.
> - For **smaller files**, it defaults to **simple in-memory processing** (`loadJSON`) for faster performance.
>
> This approach ensures optimal balance between **efficiency** and **stability**, especially when working with hundreds of thousands of records (e.g., over 850,000 for MemberProfile).

## 💡 Note on Upserts

By default, many submodels like `ChallengePrizeSet` and `Prize` use generated UUIDs as primary keys. Since these IDs don't exist in the original data files, **it's not feasible to use `upsert`** unless a unique composite constraint like:

```prisma
@@unique([challengeId, name])
```

is added in the Prisma schema. We suggest Topcoder adopts this pattern to enable reliable upserts for:

- `ChallengeMetadata`
-  `ChallengePrizeSet`
-  `Prize`
-  `ChallengeWinner`
-  `ChallengeTerm`
-  `ChallengeSkill`
- `ChallengeBilling`
- `ChallengeDiscussion`
- `ChallengeDiscussionOption`
- `ChallengeConstraint`
- `ChallengePhase`
- `ChallengePhaseConstraints`

---

### 🔎 Notable Implementation Details

- **Status Normalization**:  
  The `status` field from the original dataset contains a variety of inconsistent formats (e.g., `"Cancelled – Zero Registrations"`, `"active"`, `"Completed"`).  
  To ensure schema compatibility with the expected enums in PostgreSQL, the migration applies normalization logic that:

  - Converts values to uppercase,
  - Replaces spaces and dashes with underscores,
  - Defaults to `'NEW'` if the value is not in the predefined enum list.

  **Example transformation:**

  ```
  "Cancelled – Zero Registrations"  →  "CANCELLED_ZERO_REGISTRATIONS"
  ```

  This guarantees safe and consistent insertion of all status values into the database while respecting enum constraints.

## ✅ Validate

A validation script is included to **cross-check records between the NDJSON input and the PostgreSQL database**.

It compares fields like `name`, `typeId`, `trackId`, `status`, and `description` for each challenge, and reports:
- ✅ Valid entries (perfect match)
- ⚠️ Field mismatches (e.g., `status` normalization issues)
- ❌ Missing entries in the database

The script uses `readline` for streaming large NDJSON files efficiently, and normalizes `status` values using the same logic as the migration (e.g., converting `Cancelled – Zero Registrations` to `CANCELLED_ZERO_REGISTRATIONS`).

Run with:

```bash
node scripts/validateChallenges.js ./data/challenge-api.challenge.json

---

## 📝 Logs

Failed or skipped migrations are written to `logs/`:

- `challenge_errors.log`
- `challenge_metadata_errors.log`
- `auditlog_skipped_invalid_fk.log`

---

## 🧪 Tests

Run tests:

```bash
npm test
```

Mock data for testing is under `src/test/mocks/`.

---

✅ All primary and relational challenge models have been migrated successfully. Audit logs and attachments were excluded due to reference issues.
