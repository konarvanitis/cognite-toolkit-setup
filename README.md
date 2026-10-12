# cognite-toolkit-setup
A test repository for setting up a [Cognite Toolkit](https://docs.cognite.com/cdf/deploy/cdf_toolkit/) project. It deploys the CFIHOS oil & gas example modules from the Cognite module library to a CDF project.

## Getting Started

### Prerequisites

- Python 3.11 (see `.python-version`)
- [uv](https://docs.astral.sh/uv/)

### 1. Install dependencies

```bash
uv sync
```

This installs the dependencies defined in `pyproject.toml` (pinned in `uv.lock`), including `cognite-toolkit`, and creates a virtual environment (`.venv`) in the project folder.

### 2. Configure credentials

Copy `.env.template` to `.env` and fill in your CDF details:

```bash
cp .env.template .env
```

Alternatively, `uv run cdf auth init` creates the `.env` file interactively.

Besides the login variables, `config.dev.yaml` and `config.prod.yaml` read these variables from `.env`:

| Variable | Used for |
|---|---|
| `IDP_CLIENT_ID`, `IDP_CLIENT_SECRET` | Ingestion and function clients |
| `GROUP_SOURCE_ID` | Ingestion, entity matching and file annotation groups |
| `ADMIN_SOURCE_ID`, `CONSUMER_SOURCE_ID`, `PRODUCER_SOURCE_ID` | Project foundation groups |
| `OWNER_GROUP_SOURCE_ID`, `READ_GROUP_SOURCE_ID` | CFIHOS data model groups |

`.env` is git-ignored. Never commit it.

### 3. Set the target project

Set `environment.project` in `config.dev.yaml` (currently `test-dev`) and `config.prod.yaml` (currently `<my-project-prod>`) to your CDF project names.

### 4. Build and deploy

All Toolkit commands run through uv:

```bash
uv run cdf auth verify                         # check that your credentials have the required access
uv run cdf build --config-yaml config.dev.yaml # build the modules into ./build
uv run cdf deploy --dry-run                    # preview the changes
uv run cdf deploy                              # deploy to CDF
```

Use `config.prod.yaml` to build for prod.

## Modules

The modules in `modules/` come from the Cognite library (`[library.cognite]` in `cdf.toml`):

| Folder | Contents |
|---|---|
| `common` | `cdf_project_foundation` (groups, spaces, data sets), `cdf_ingestion` (ingestion workflow) |
| `datamodels` | `cfihos_oil_and_gas_extension`, `cfihos_oil_and_gas_extension_search` |
| `sourcesystem` | `cdf_pi_data_dump`, `cdf_sap_data_dump`, `cdf_sharepoint_data_dump` (example source data) |
| `contextualization` | `cdf_file_annotation`, `cdf_entity_matching` |

Module variables are set under `variables.modules` in `config.<env>.yaml`.

## Additional notes for developers

### Add new libraries as needed

```bash
uv add pandas numpy
```

or if only required for development

```bash
uv add --dev pytest
```

### Upgrade the Toolkit

```bash
uv add "cognite-toolkit>=<new-version>"
uv run cdf modules upgrade
```

`cdf modules upgrade` updates the module version in `cdf.toml`. Do not edit it by hand.
