---
name: create-project
description: "Create new DCM projects from scratch. Triggers: new project, create dcm project, set up dcm, start from scratch, build infrastructure"
---

# Create New DCM Project

## Overview

This sub-skill guides you through creating a new DCM project from scratch, including project registration in Snowflake, local file structure creation, and initial definition writing.

## Prerequisites

- Snowflake connection name
- Target database and schema for the project (must exist)
- Understanding of what infrastructure the user wants to define

## Step-by-Step Workflow

### Step 1: Gather Project Requirements

**Ask the user:**

1. **Project identifier**: What should the project be named?

   - Format: `DATABASE.SCHEMA.PROJECT_NAME`
   - Example: `MY_DB.PROJECTS.ANALYTICS_PIPELINE`
   - **⚠️ CRITICAL**: The DATABASE and SCHEMA in the identifier are the *parent containers* where the project lives. You CANNOT define these containers in the project itself. For example, if the project is `MY_DB.MY_SCHEMA.MY_PROJECT`, you cannot use `DEFINE DATABASE MY_DB` or `DEFINE SCHEMA MY_DB.MY_SCHEMA`. You can only define objects *inside* `MY_DB.MY_SCHEMA`.

The user can provide the response in a freeform manner, ask clarification if needed.

2. **What infrastructure to define?**

   - Databases and schemas
   - Tables (source tables, staging tables)
   - Dynamic tables (transformations)
   - Views (consumption layer)
   - Internal stages (for data loading/unloading)
   - External stages (S3, Azure, GCS - requires POST_HOOK)
   - Tasks (scheduled operations, ETL orchestration)
   - Warehouses
   - Roles and grants
   - Data quality expectations
   - etc.

The user might provide some of this information in the initial prompt, so ask clarification questions as needed.

**⚠️ Stage-Specific Guidance:**
   - Internal stages (no URL) → Use `DEFINE STAGE` in infrastructure.sql
   - External stages (S3/Azure/GCS with URL) → Use `ATTACH POST_HOOK`

2.5. **⚠️ If roles/grants are needed, clarify:**

   - **Role hierarchy pattern?** (e.g., ADMIN → DEVELOPER → ANALYST)
   - **Warehouse access?** Which roles need warehouse usage?
     - ⚠️ Warehouse grants CANNOT go to database roles (Snowflake constraint)
     - Will need an account role for warehouse access
   - **Any privileged grants?** (ON ACCOUNT, IMPORTED PRIVILEGES, etc.)
     - These are unsupported in DCM and need post-deployment manual application
   - **User assignments?** Which users get which roles?

   **→ Load** [../dcm-roles-and-grants/SKILL.md](../dcm-roles-and-grants/SKILL.md) for recommended patterns.

3. **Multi-environment support?**
   - Will this deploy to DEV/PROD separately?
   - What differs between environments? (database names, warehouse sizes, users)

### Step 2: Verify Prerequisites

Check that the target database and schema exist:

```sql
SHOW DATABASES LIKE '<database>';
SHOW SCHEMAS IN DATABASE <database> LIKE '<schema>';
```

If they don't exist, prompt the user:

> "The database/schema for your DCM project doesn't exist. Would you like me to create them first, or would you prefer to create them manually?"

### Step 3: Create DCM Project in Snowflake

Register the project:

```bash
snow dcm create <DATABASE.SCHEMA.PROJECT_NAME> -c <connection>
```

Expected output: Confirmation that the project was created.

### Step 4: Create Local Project Structure

Create the recommended directory structure:

Refer to [../reference/project_structure.md](../reference/project_structure.md) for the best practice directory structure.

### Step 5: Create manifest.yml

Use ONLY the fields shown in these examples. The deployment target is specified via CLI commands (`snow dcm plan DB.SCHEMA.PROJECT`), not in the manifest.

**Without configurations (single environment):**

```yaml
manifest_version: 1
include_definitions:
  - definitions/.*
type: DCM_PROJECT
```

**With configurations (multi-environment):**

```yaml
manifest_version: 1
include_definitions:
  - definitions/.*
type: DCM_PROJECT

configurations:
  DEV:
    env: "DEV"
    wh_size: "X-SMALL"
    users:
      - "DEV_USER"
  PROD:
    env: "PROD"
    wh_size: "LARGE"
    users:
      - "PROD_SERVICE_USER"
```

### Step 6: Clarify Object Definitions

Before writing definitions, **provide a proposed structure and get confirmation from the user**

### Step 7: Write Definition Files

Refer to [../reference/project_structure.md](../reference/project_structure.md) for the best practice directory structure.

Refer to [../reference/syntax.md](../reference/syntax.md) for the syntax of the definitions.

### Step 8: Validate with Analyze

Run analyze to validate the project:

```bash
snow dcm analyze <identifier> -c <connection> \
    --configuration <config> \
    --output-path ./out/analyze
```

#### ⚠️ CRITICAL: Read and Parse the Output

**You MUST read and parse `out/analyze/analyze_output.json`.**

For detailed instructions on reading output files, see: [Parent SKILL.md - Critical: Reading Output Files](../SKILL.md#️-critical-reading-output-files)

### Step 9: Fix Any Errors

Common issues and fixes:

| Error                  | Cause                     | Solution                                       |
| ---------------------- | ------------------------- | ---------------------------------------------- |
| "Unknown object"       | Missing dependency        | Define the referenced object or check spelling |
| "Syntax error"         | Invalid DCM syntax        | Check DEFINE syntax matches CREATE equivalent  |
| "Duplicate definition" | Same object defined twice | Remove duplicate or consolidate                |
| "Invalid identifier"   | Bad naming                | Use fully qualified names                      |

### Step 10: Next Steps

Once analysis passes without errors:

1. **Ask user**: "The project has been created and validated. Would you like to:"

   - Run a plan to see what will be created?
   - Preview any data in existing objects? Only ask if the project contains previewable objects.

2. **Load** the deploy-project sub-skill from [../deploy-project/SKILL.md](../deploy-project/SKILL.md) if user wants to deploy

## Tips

### Naming Conventions

- Use UPPERCASE for Snowflake object names (matches Snowflake default)
- Use `{{env}}` or similar variables for environment-specific naming
- Be consistent with prefixes/suffixes

### File Organization

Keep related objects together:

- All tables in one file, or
- Group by business domain (sales.sql, marketing.sql, etc.)

### Configuration Variables

Common variables to define:

- `env`: Environment identifier (DEV, PROD)
- `wh_size`: Warehouse size
- `users`: List of users for grants
- `teams`: List of team/schema names
