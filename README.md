  # Topcoder Resources Data Migration Tool
  
  This tool is designed to **migrate data from DynamoDB (JSON format) to PostgreSQL** using **Prisma ORM**. It covers five key models of the Topcoder Resources API:
  
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

### Skipped Models (due to invalid or missing references):

- `AuditLog`
- `Attachment`

Skipped because the referenced `challengeId`s are not present for both in `challenge-api.challenge.json`.

---
  
  ## 📦 Technologies Used
  - **Node.js** (backend scripting)
  - **Prisma ORM** (PostgreSQL schema management)
  - **PostgreSQL 16.3** (Dockerized database)
  - **Docker & Docker Compose** (for DB setup)
  - **stream-json / readline** (for streaming JSON migration)
  - **Jest** (unit testing framework)
  
  ## ⚙️ Environment Configuration
  Create a `.env` file in the root directory:
  
  ```env
  DATABASE_URL="postgresql://postgres:postgres@localhost:5432/challengedb"
  CREATED_BY="challenge-api-db-migration"
  ```
  
  > The `CREATED_BY` field can be overridden at runtime:
  ```bash
  CREATED_BY=eduardo node src/index.js member-stats ./data/MemberStats_test.json
  ```
  
  ## 🚀 How to Run
  
  This tool expects a running PostgreSQL instance defined in `docker-compose.yml`.
  
  1. Clone the repo and install dependencies:
  
  ```bash
  npm install
  ```
  
  2. Start PostgreSQL with Docker Compose:
  
  ```bash
  docker-compose up -d
  ```
  
  To tear it down completely (including the volume):
  
  ```bash
  docker-compose down -v
  ```
  
  > The database runs on port `5432` with credentials `postgres:postgres`, and is mapped to `resourcesdb`.
  
  3. Push the Prisma schema to the database:
  
  ```bash
  npx prisma db push
  ```
  
  4. Run a migration step (with optional file override):
  
  ```bash
  node src/index.js member-stats
  node src/index.js resources ./data/challenge-api.resources.json
  ```
  
  You can override the default `createdBy` value:
  
  ```bash
  CREATED_BY=my-migrator node src/index.js member-profiles
  ```
  
  ## 🧩 Available Migration Steps
  
  | Step                                | Auto Strategy | Description                                                                                       |
  |-------------------------------------|---------------|---------------------------------------------------------------------------------------------------|
  | `member-profiles`                  | ✅            | Auto strategy: uses `stream-json` (batch) for files larger than 3MB, and `loadJSON` (simple) otherwise |
  | `member-stats`                     | ✅            | Auto strategy: uses `stream-json` (batch) for files larger than 3MB, and `loadJSON` (simple) otherwise |
  | `resource-roles`                   | ❌            | Simple in-memory migration using `loadJSON`, not expected to be large                             |
  | `resource-role-phase-dependencies` | ❌            | Simple in-memory migration using `loadJSON`, not expected to be large                             |
  | `resources`                        | ✅            | Auto strategy for NDJSON files: uses `readline` + batch for files > 3 MB, otherwise simple line-by-line       |

  > ⚙️ **Why Auto Strategy?**
>
> For models that involve large datasets (`member-profiles`, `member-stats`, and `resources`), the tool implements an **automatic selection strategy** based on file size:
> - If the input file is **larger than 3 MB**, the migration runs in **batch mode using streaming (e.g., `stream-json` or `readline`)** to reduce memory usage.
> - For **smaller files**, it defaults to **simple in-memory processing** (`loadJSON`) for faster performance.
>
> This approach ensures optimal balance between **efficiency** and **stability**, especially when working with hundreds of thousands of records (e.g., over 850,000 for MemberProfile).
  
  ### 📁 Default Input Files per Migration Step
  
  The following files are used by default for each step, unless a custom path is provided via the CLI:
  
  | Step                                | Default File Path                                             |
  |-------------------------------------|----------------------------------------------------------------|
  | `member-profiles`                  | `./data/MemberProfile_dynamo_data.json`                       |
  | `member-stats`                     | `./data/MemberStats_dynamo_data.json`                         |
  | `resource-roles`                   | `./data/ResourceRole_dynamo_data.json`                        |
  | `resource-role-phase-dependencies` | `./data/ResourceRolePhaseDependency_dynamo_data.json`         |
  | `resources`                        | `./data/Resource_data.json` ← requires NDJSON format          |
  
  💡 **Note:** If you're using the original ElasticSearch export file (`challenge-api.resources.json`) provided in the forum ([link here](https://drive.google.com/file/d/1F8YW-fnKjn8tt5a0_Z-QenZIHPiP3RK7/view?usp=sharing)), you must explicitly provide its path when running the migration:
  
  ```bash
  node src/index.js resources ./data/challenge-api.resources.json
  ```
  
  ## 📒 Error Logs
  All failed migrations are logged under the `logs/` folder by model:
  
  - `logs/memberprofile_errors.log` ← from `MemberProfile_dynamo_data.json` *(7 migrations failed)*
  - `logs/memberstats_errors.log` ← from `MemberStats_dynamo_data.json` *(1 migration failed)*
  - `logs/rrpd_errors.log` ← from `ResourceRolePhaseDependency_dynamo_data.json` *(17 migrations failed)*
  
  > ✅ Most migrations complete successfully. Errors are logged for further review and debugging.
  
  ## ✅ Verification
  You can verify successful migration with simple SQL queries, for example:
  ```sql
  SELECT COUNT(*) FROM "MemberProfile";
  SELECT COUNT(*) FROM "Resource";
  ```
  To connect:
  ```bash
  docker exec -it resources_postgres psql -U postgres -d resourcesdb
  ```
  
  ## 📸 Screenshots
  See `/docs/` for evidence of a fully mounted database.
  ![Screenshot from 2025-04-14 16-58-20](https://github.com/user-attachments/assets/8fb66fb8-3db1-4b51-bb29-c1db7b207689)
  
  ## 🧪 Testing
  
  Run all test suites with:
  
  ```bash
  npm test
  ```
  
  Each migrator has a corresponding unit test with mock input files under `src/test/mocks/`. Jest is used as the testing framework.
  
  ---
  
### 📂 Data Files Not Included

The official DynamoDB dataset files provided in the forum (e.g., `MemberProfile_dynamo_data.json`, `challenge-api.resources.json`, etc.) are **not included** in this submission due to size constraints.

Please download them manually from the official challenge forum and place them under the `/data/` directory.

🔗 [Official Data Files (Google Drive)](https://drive.google.com/file/d/1F8YW-fnKjn8tt5a0_Z-QenZIHPiP3RK7/view?usp=sharing)

> 🧪 This project **includes lightweight mock data files** under `src/test/mocks/` for testing purposes and sample execution. Full data is only required for production migration.
  
  ---
  
  ✅ All requirements of the challenge have been implemented, including logs, unit tests, schema adherence, and configurability.
  
