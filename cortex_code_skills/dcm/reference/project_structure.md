# DCM Project Structure Reference

This document describes the structure of DCM (Database Change Management) projects, including the manifest file, definition patterns, and configuration management.

> **Note for Cortex Agents**: This document covers project structure only. For DCM definition syntax (`DEFINE` statements, grants, data quality, etc.), see `reference/syntax.md`.

---

## Project Overview

A DCM project is a directory containing:

1. **`manifest.yml`** — The project manifest (required)
2. **Definition files** — SQL files with `DEFINE` statements (`.sql` files)

### Recommended Structure

The simplest and recommended project layout uses a single `definitions/` folder:

```
my_project/
├── manifest.yml
└── definitions/
    ├── <definition_name>.sql
    ├── <definition_name>.sql
    ├── <definition_name>.sql
```

> **Best Practice**: Keep the structure with a single `definitions/` folder containing SQL files. Nest files logically within the `definitions/` folder, for example by purpose ('raw', 'analytics') or areas ('product', 'sales', 'marketing', 'finance', 'hr', 'it', 'sales'). In simple cases, prefer a flat structure under a single `definitions/` folder.

### Alternative Structures

More complex structures are possible (nested folders, multiple definition directories), they add complexity without significant benefit for most use cases:

```
# More complex example (from TPCDI_PROJECT) — works but not recommended for most simple projects
project/
├── manifest.yml
└── definitions/
    ├── expectations.sql
    ├── grants.sql
    ├── high_level_objects.sql
    ├── TPCDI_ODS/
    │   └── DYNAMIC_TABLES.sql
    ├── TPCDI_STG/
    │   ├── DYNAMIC_TABLES.sql
    │   └── TABLES.sql
    └── TPCDI_WH/
        ├── DYNAMIC_TABLES.sql
        └── VIEWS.sql
```

---

## The Manifest File (`manifest.yml`)

The manifest file is the heart of a DCM project. It defines which files to include, how to configure environments, and project metadata.

### Complete Schema

```yaml
# Required: Manifest version (currently 1)
manifest_version: 1

# Required: Patterns matching definition files to include
include_definitions:
  - definitions/.*

# REQUIRED: Project type identifier, must match exactly
type: DCM_PROJECT

# Optional: Environment configurations exposing Jinja variables
configurations:
  DEV:
    db: "DEV"
    wh_size: "X-SMALL"
  PROD:
    db: "PROD"
    wh_size: "LARGE"
```

### Required Fields

> **Manifest Schema:** Only the fields documented below are valid. Any other fields will cause "is not defined in the schema" errors. The deployment target is specified via CLI commands (`snow dcm plan DB.SCHEMA.PROJECT`), not in the manifest.

#### `manifest_version`

**Type**: `number`

The manifest schema version. Currently, only version `1` is supported.

```yaml
manifest_version: 1
```

#### `include_definitions`

**Type**: `array` of strings (unique, non-empty)

Java regular expression patterns that match definition files to include in the project. Patterns are evaluated relative to the project root.

```yaml
include_definitions:
  - definitions/.*
```

> **Best Practice**: Use the simple pattern `definitions/.*` to include all files in a single `definitions/` folder. This is the recommended default.

**Pattern Examples**:

| Pattern                  | Matches                                        |
| ------------------------ | ---------------------------------------------- |
| `definitions/.*`         | All files in `definitions/` and subdirectories |
| `definitions/[^/]*\.sql` | Only `.sql` files directly in `definitions/`   |
| `definitions/tables/.*`  | All files under `definitions/tables/`          |

> **Note**: These are Java regex patterns, not glob patterns. The `.*` means "any characters" (regex), not "any files" (glob).

#### `type`

**Type**: `string` (case-insensitive, must match `DCM_PROJECT`)

Identifies this as a DCM project. MUST always be set to `DCM_PROJECT`.

```yaml
type: DCM_PROJECT
```

### Optional Fields

#### `configurations`

**Type**: `object` (Record of configuration names to variable mappings)

Defines named configurations (typically environments) that expose Jinja template variables to definition files. See [Configurations](#configurations-and-environments) for detailed usage.

```yaml
configurations:
  DEV:
    db: "DEV"
    wh_size: "X-SMALL"
  PROD:
    db: "PROD"
    wh_size: "LARGE"
```

## Configurations and Environments

Configurations are the primary mechanism for managing environment-specific values (dev, test, prod, etc.). Each configuration defines a set of variables that become available as Jinja template variables in definition files.

### Basic Configuration Structure

```yaml
configurations:
  CONFIGURATION_NAME:
    variable_name: value
    another_variable: "string value"
    numeric_value: 100
    boolean_flag: true
    list_variable:
      - "item1"
      - "item2"
```

### Supported Value Types

| Type    | Example             | Usage in Jinja         |
| ------- | ------------------- | ---------------------- |
| String  | `db: "PROD"`        | `{{db}}`               |
| Number  | `timeout: 300`      | `{{timeout}}`          |
| Boolean | `enabled: true`     | `{% if enabled %}`     |
| Array   | `users: ["A", "B"]` | `{% for u in users %}` |

### Common Configuration Patterns

#### Environment-Based Sizing

```yaml
configurations:
  DEV:
    db: "DEV"
    wh_size: "X-SMALL"

  TEST:
    db: "TEST"
    wh_size: "SMALL"

  PROD:
    db: "PROD"
    wh_size: "LARGE"
```

**Usage in definitions**:

```sql
DEFINE WAREHOUSE PROJECT_WH_{{db}}
WITH
    WAREHOUSE_SIZE = '{{wh_size}}'
    AUTO_SUSPEND = 300;
```

**Result for DEV**: Creates `PROJECT_WH_DEV` with size `X-SMALL`  
**Result for PROD**: Creates `PROJECT_WH_PROD` with size `LARGE`

#### Environment Suffixes

Use suffixes to create distinct object names per environment:

```yaml
configurations:
  DEV:
    env_suffix: "_DEV"

  PROD:
    env_suffix: "" # No suffix in production
```

**Usage in definitions**:

```sql
DEFINE DATABASE MY_PROJECT{{env_suffix}};
DEFINE SCHEMA MY_PROJECT{{env_suffix}}.RAW;
```

**Result for DEV**: Creates `MY_PROJECT_DEV.RAW`  
**Result for PROD**: Creates `MY_PROJECT.RAW`

#### User and Role Management

```yaml
configurations:
  DEV:
    project_owner_role: "DCM_DEVELOPER"
    users:
      - "DEV_USER"

  PROD:
    project_owner_role: "DCM_PROD_DEPLOYER"
    users:
      - "GITHUB_ACTIONS_SERVICE_USER"
      - "ADMIN_USER"
```

**Usage in definitions**:

```sql
{% for user_name in users %}
    GRANT ROLE PROJECT_READ TO USER {{user_name}};
{% endfor %}
```

#### Team-Based Schemas

```yaml
configurations:
  DEV:
    teams:
      - "DEV_TEAM"

  PROD:
    teams:
      - "Marketing"
      - "Finance"
      - "HR"
      - "Sales"
```

**Usage in definitions**:

```sql
{% for team in teams %}
    DEFINE SCHEMA MY_DB.{{ team | upper }};
{% endfor %}
```

### Complete Configuration Example

```yaml
manifest_version: 1

include_definitions:
  - definitions/.*

type: DCM_PROJECT

configurations:
  DEV:
    db: "DEV"
    wh_size: "X-SMALL"
    project_owner_role: "DCM_DEVELOPER"
    sample_size: "5"
    users:
      - "DEV_USER"
    teams:
      - "DEV_TEAM"

  TEST:
    db: "TEST"
    wh_size: "SMALL"
    project_owner_role: "DCM_DEVELOPER"
    sample_size: "10"
    users:
      - "DEV_USER"
      - "QA_USER"
    teams:
      - "TEST_TEAM"

  PROD:
    db: "PROD"
    wh_size: "LARGE"
    project_owner_role: "DCM_PROD_DEPLOYER"
    sample_size: "100"
    users:
      - "GITHUB_ACTIONS_SERVICE_USER"
    teams:
      - "Marketing"
      - "Finance"
      - "HR"
      - "IT"
      - "Sales"
```

---

## Jinja Templating in Definitions

Configuration variables are exposed to definition files as Jinja template variables. While DCM supports the full Jinja2 templating language, keeping templates simple is strongly recommended.

### Simple Variable Substitution (Preferred)

```sql
DEFINE DATABASE MY_PROJECT_{{db}};

DEFINE WAREHOUSE MY_WH_{{db}}
WITH
    WAREHOUSE_SIZE = '{{wh_size}}';
```

### Loops for Lists

```sql
{% for user_name in users %}
    GRANT ROLE PROJECT_READ TO USER {{user_name}};
{% endfor %}
```

### Conditionals (Use Sparingly)

```sql
{% for team in teams %}
    DEFINE SCHEMA MY_DB.{{ team | upper }};

    {% if team == 'HR' %}
        DEFINE TABLE MY_DB.{{ team | upper }}.EMPLOYEES (
            NAME VARCHAR,
            ID INT
        );
    {% endif %}
{% endfor %}
```

### Jinja Best Practices

| Do                                     | Don't                              |
| -------------------------------------- | ---------------------------------- |
| Use simple `{{variable}}` substitution | Create deeply nested logic         |
| Keep loops straightforward             | Chain multiple conditionals        |
| Use macros for repeated patterns       | Over-engineer with complex filters |
| Make definitions readable              | Sacrifice clarity for DRY          |

> **Warning**: While Jinja is powerful, excessive templating makes definitions hard to read and debug. If you find yourself writing complex Jinja logic, consider whether simpler approaches (like separate definition files per environment) might be clearer.

---

## Definition Files

Definition files are SQL files containing DCM `DEFINE` statements. They describe the desired state of Snowflake objects.

### File Organization

Organize definition files by logical grouping:

| File                        | Contents                                   |
| --------------------------- | ------------------------------------------ |
| `database.sql` or `raw.sql` | Databases, schemas, base tables            |
| `analytics.sql`             | Dynamic tables, analytical transformations |
| `serve.sql`                 | Views for consumption                      |
| `access.sql`                | Roles, grants, permissions                 |
| `expectations.sql`          | Data metric functions, data quality rules  |

### Example: Simple Project

**`definitions/database.sql`**:

```sql
DEFINE DATABASE MY_PROJECT_{{db}};
DEFINE SCHEMA MY_PROJECT_{{db}}.RAW;
DEFINE SCHEMA MY_PROJECT_{{db}}.ANALYTICS;
```

**`definitions/tables.sql`**:

```sql
DEFINE TABLE MY_PROJECT_{{db}}.RAW.CUSTOMERS (
    CUSTOMER_ID NUMBER,
    NAME VARCHAR,
    EMAIL VARCHAR
)
CHANGE_TRACKING = TRUE;

DEFINE TABLE MY_PROJECT_{{db}}.RAW.ORDERS (
    ORDER_ID NUMBER,
    CUSTOMER_ID NUMBER,
    ORDER_DATE DATE,
    AMOUNT NUMBER(10,2)
)
CHANGE_TRACKING = TRUE;
```

**`definitions/access.sql`**:

```sql
DEFINE WAREHOUSE MY_PROJECT_WH_{{db}}
WITH
    WAREHOUSE_SIZE = '{{wh_size}}'
    AUTO_SUSPEND = 300;

DEFINE ROLE MY_PROJECT_{{db}}_READ;

{% for user_name in users %}
    GRANT ROLE MY_PROJECT_{{db}}_READ TO USER {{user_name}};
{% endfor %}

GRANT USAGE ON DATABASE MY_PROJECT_{{db}} TO ROLE MY_PROJECT_{{db}}_READ;
GRANT USAGE ON SCHEMA MY_PROJECT_{{db}}.RAW TO ROLE MY_PROJECT_{{db}}_READ;
GRANT SELECT ON ALL TABLES IN DATABASE MY_PROJECT_{{db}} TO ROLE MY_PROJECT_{{db}}_READ;
```

## Project Structure Best Practices

### For New Projects

1. **Start with the simple structure**:

   ```
   project/
   ├── manifest.yml
   └── definitions/
       └── (all .sql files here)
   ```

2. **Use `definitions/.*` as your include pattern**

3. **Define configurations for your environments** (at minimum: DEV and PROD)

4. **Keep Jinja simple** — prefer explicit over clever

### Naming Conventions

| Convention                      | Example                            | Purpose                             |
| ------------------------------- | ---------------------------------- | ----------------------------------- |
| Environment suffix in names     | `MY_DB_{{db}}`                     | Distinguish objects per environment |
| Uppercase for Snowflake objects | `DEFINE TABLE MY_DB.RAW.CUSTOMERS` | Match Snowflake conventions         |
| Descriptive file names          | `access.sql`, `expectations.sql`   | Easy navigation                     |

### Configuration Variable Naming

| Variable             | Description            | Example Values         |
| -------------------- | ---------------------- | ---------------------- |
| `db` or `env`        | Environment identifier | `"DEV"`, `"PROD"`      |
| `env_suffix`         | Object name suffix     | `"_DEV"`, `""`         |
| `wh_size`            | Warehouse size         | `"X-SMALL"`, `"LARGE"` |
| `users`              | User list for grants   | `["USER1", "USER2"]`   |
| `teams`              | Team/schema list       | `["Finance", "HR"]`    |
| `project_owner_role` | Top-level role         | `"DCM_DEVELOPER"`      |

---

## Summary

| Component             | Required         | Purpose                               |
| --------------------- | ---------------- | ------------------------------------- |
| `manifest.yml`        | Yes              | Project configuration and metadata    |
| `include_definitions` | Yes              | Patterns for finding definition files |
| `configurations`      | No (recommended) | Environment-specific variables        |
| `definitions/` folder | Yes              | SQL files with DEFINE statements      |

**The simplest valid project**:

```yaml
# manifest.yml
manifest_version: 1
include_definitions:
  - definitions/.*
type: DCM_PROJECT
```

```sql
-- definitions/main.sql
DEFINE DATABASE MY_PROJECT;
DEFINE SCHEMA MY_PROJECT.RAW;
```

For syntax details on `DEFINE` statements, grants, and data quality rules, see `reference/syntax.md`.
