# Topcoder Challenge API – Data Migration Tool

This tool migrates data from **ElasticSearch (NDJSON)** and **DynamoDB-style JSON** to **PostgreSQL**, using **Prisma ORM**.

It supports the complete Challenge API schema (as of April 2025) and is optimized for performance, correctness, and configurability.

---

## ✅ Supported Models

The following models are migrated when valid data is present:

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

### Skipped Models (due to invalid or missing references):

- `AuditLog`
- `Attachment`

Skipped because none of the referenced `challengeId`s exist in `challenge-api.challenge.json`.

---

## 📦 Technologies Used

- **Node.js** – migration scripts
- **Prisma ORM** – schema modeling and database operations
- **PostgreSQL 16.3** – managed via Docker Compose
- **stream-json / readline** – efficient NDJSON parsing
- **cli-progress** – progress bar for large migrations
- **Jest** – testing framework

---

## ⚙️ Environment Setup

Create a `.env` file in the root directory:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/challengedb"
CREATED_BY="challenge-api-db-migration"
```

Override `CREATED_BY` during execution if needed:

```bash
CREATED_BY=my-script node src/index.js challenges
```

---

## 🚀 Running Migrations

1. Install dependencies:

```bash
npm install
```

2. Start PostgreSQL:

```bash
docker-compose up -d
```

3. Push the Prisma schema:

```bash
npx prisma db push
```

4. Run individual migrations:

```bash
node src/index.js challenge-types
node src/index.js challenges ./data/challenges_sample.ndjson
```

5. (Optional) Run all in sequence:

```bash
node src/index.js migrate-all
```

---

## 🧩 Available Migration Steps

| Step                           | Description                                                  |
|--------------------------------|--------------------------------------------------------------|
| `challenge-types`             | Imports `ChallengeType` definitions                         |
| `challenge-tracks`            | Imports `ChallengeTrack` definitions                        |
| `phases`                      | Imports global `Phase` definitions                          |
| `timeline-templates`          | Imports `TimelineTemplate` base structures                  |
| `challenge-timeline-templates`| Links challenge types to timeline templates                 |
| `attachments`                 | Skipped – invalid references                                |
| `audit-logs`                  | Skipped – invalid references                                |
| `challenges`                  | Full challenge with all submodels (skills, winners, etc.)   |

---

## 🧪 Verification Strategy

Each migration is **verifiable** via:

- **SQL count comparisons**
- **Optional field-by-field validation scripts**
- **Upsert logic** ensures incremental updates are supported

Example:

```sql
SELECT COUNT(*) FROM "Challenge";
SELECT * FROM "ChallengeMetadata" WHERE challengeId = 'xyz';
```

Validation scripts can be extended to compare source NDJSON vs PostgreSQL row-by-row if needed.

---

## 📂 Logs & Debugging

Each model writes migration errors to its respective log under `/logs`:

- `challenge_errors.log`
- `challenge_metadata_errors.log`
- `auditlog_errors.log`
- etc.

Skipped records (e.g., invalid foreign keys) are also logged separately.

---

## 🧪 Testing

```bash
npm test
```

Mock files are available in `src/test/mocks/` for unit testing migrators.

---

## 📸 Sample Output

Terminal progress:

```
📦  Migrating challenges...
📊  Processed 1,000 records...
✅  Challenge migration finished: 1,983 success, 17 failed
```

PostgreSQL (via Docker):

```bash
docker exec -it challenge_postgres psql -U postgres -d challengedb
```

---

## 📁 Notes on Data Files

The original NDJSON files (e.g., `challenge-api.challenge.json`) are **not included** due to file size.

Please download them manually from the official forum and place them in `/data`.

---

✅ This project fulfills all requirements including:

- Full Prisma schema
- Auto and batch migration
- Skipped model logging
- Validation support
- Modular and testable code
