# A Practical Guide to Setting Up Cognite Toolkit

This guide walks you through setting up a new Cognite Toolkit project with [uv](https://docs.astral.sh/uv/), from creating the local folder to deploying your first configuration to CDF.

You do not need to be a Toolkit expert to follow it. Each section explains both **what to do** and **why it matters**.

> **In simple terms:** you write and manage CDF resource definitions in `modules/`, use `cdf build` to prepare them, and use `cdf deploy` to send them to your CDF project.

## 1. Before You Start

Make sure the following tools and project details are available before you begin:

- Python 3.11 or later (uv can install it for you with `uv python install 3.11`)
- uv
- Git
- A provisioned CDF project
- The CDF project name and cluster
- Identity provider details, such as:
  - Microsoft Entra ID tenant ID
  - Application/client ID when required
  - CDF group object IDs when required by the selected modules

You can quickly check the installed tools with:

```sh
python --version
uv --version
git --version
```

### Note for Windows users

If `python` is not recognized, try:

```powershell
py --version
```

Windows Terminal or PowerShell is recommended because the Toolkit setup includes interactive menus.

## 2. Create a Home for the Toolkit Project

Start by creating a folder for your organization's Toolkit configuration. This folder can hold configurations for all of your environments, such as development, test, and production.

```sh
mkdir <organization-folder>
cd <organization-folder>
```

Stay in this directory while following the rest of the guide. Toolkit commands look for files such as `cdf.toml` and `.env` in the current directory.

## 3. Set Up uv

Initialize uv in the project folder and tell it that the project requires Python 3.11 or later:

```sh
uv init --bare --name cdf-toolkit --python 3.11
```

This creates only `pyproject.toml`, which uv uses to track Python requirements and dependencies.

The name `cdf-toolkit` is only the local project name. It does not need to match your CDF project name.

uv finds a compatible Python installation automatically and downloads one if none is installed. If several Python versions are installed and you want the project to use a specific one, pin it:

```sh
uv python pin 3.11
```

This writes the version to `.python-version`, which uv uses when it creates the virtual environment.

## 4. Install Cognite Toolkit

Now install Cognite Toolkit into the project's virtual environment:

```sh
uv add cognite-toolkit
```

uv creates the `.venv` folder and the `uv.lock` file the first time you add a dependency.

Check that the installation worked:

```sh
uv run cdf --version
```

> **Why use `uv run`?** It guarantees that the command uses the Toolkit version installed for this project. Running only `cdf` could accidentally use an older global installation.

## 5. Create the Initial Toolkit Files

Start the guided setup:

```sh
uv run cdf init
```

The wizard presents several tasks. Work through them in order: create the Toolkit configuration, set up authentication, and initialize the modules.

### Create `cdf.toml`

Select **Create toml file** and press Enter to create `cdf.toml` in the current folder.

Think of `cdf.toml` as the main settings file for the Toolkit CLI. It controls:

- Default environment
- Modules version
- Enabled plugins
- Module libraries

The module version is managed by Toolkit. Avoid editing it by hand.

### Connect Toolkit to CDF

Select **Authentication** in the wizard. You can also start the authentication setup separately:

```sh
uv run cdf auth init
```

Choose the sign-in method that matches how you plan to use Toolkit:

- `session`: Signs in through Cognite's identity provider and keeps a refreshable session (recommended by Toolkit 0.8 for local development)
- `device_code`: Signs in through a browser on any device, with no service principal needed
- `interactive`: Opens a browser directly for user sign-in
- `client_credentials`: Recommended for CI/CD and service accounts
- `token`: Uses an existing OAuth2 token

For day-to-day local development, `device_code` is usually the easiest choice. The terminal gives you a code, and you complete the sign-in in your browser.

During setup, enter the requested CDF cluster, project name, tenant ID, and any other identity provider details.

Toolkit saves the local authentication settings in `.env`.

> **Keep `.env` private:** this file can contain credentials and secrets, so never commit it to Git or share it in chat or email.

For device-code authentication, the generated values are similar to:

```dotenv
LOGIN_FLOW=device_code
CDF_CLUSTER=<cdf-cluster>
CDF_PROJECT=<cdf-project>
IDP_TENANT_ID=<tenant-id>
```

For client-credentials authentication, additional values are required:

```dotenv
LOGIN_FLOW=client_credentials
CDF_CLUSTER=<cdf-cluster>
CDF_PROJECT=<cdf-project>
IDP_CLIENT_ID=<client-id>
IDP_CLIENT_SECRET=<client-secret>
IDP_TENANT_ID=<tenant-id>
```

The `PROVIDER` variable selects the identity provider. The default is `entra_id` (Microsoft Entra ID), which needs `IDP_TENANT_ID`. With `PROVIDER=cdf` (Cognite's own identity provider, used by this repository's `.env.template`), Toolkit sets the token URL itself and no tenant ID is needed. For `auth0` or `other` providers, set `IDP_TOKEN_URL` instead of `IDP_TENANT_ID`.

### Choose the modules

Select **Modules** in the wizard, or initialize them separately:

```sh
uv run cdf modules init .
```

You can start with:

- A Cognite deployment pack, such as Foundation
- Individual modules
- An empty module structure for a custom project

Use the arrow keys to move through the menu, Space to select an item, and Enter to confirm.

After initialization, you will normally see:

```text
cdf.toml
config.dev.yaml
modules/
```

You can add `config.test.yaml` and `config.prod.yaml` later when those environments are ready.

## 6. Put the Project Under Version Control

Initialize Git, then ask Toolkit to add its recommended repository files:

```sh
git init .
uv run cdf repo init
```

Before committing anything, confirm that `.gitignore` excludes at least:

```gitignore
.env
build/
```

Commit `pyproject.toml`, `uv.lock` and `.python-version`, but not `.venv/`. This helps everyone on the team install the same dependency versions.

## 7. Confirm That Everything Works

First, ask Toolkit to show the configuration it found:

```sh
uv run cdf about
```

Then verify that your credentials can access the target CDF project:

```sh
uv run cdf auth verify
```

If verification fails, check these common causes:

- The CDF project name
- The CDF cluster
- Tenant and client IDs
- Whether the signed-in user or service principal belongs to the required CDF groups
- Whether the required CDF capabilities have been assigned

## 8. Tell Toolkit What to Deploy

Open `config.dev.yaml` and connect the local configuration to your development CDF project:

```yaml
environment:
  name: dev
  project: <cdf-development-project>
  validation-type: dev
  selected:
    - modules/

variables: {}
```

Before saving the file, make sure:

1. `project` contains the exact CDF project name.
2. `selected` contains the modules to build and deploy.
3. All module variables have valid values.
4. No `<change_me>` placeholders remain.

Keep one configuration file for each CDF environment:

```text
config.dev.yaml
config.test.yaml
config.prod.yaml
```

> **Do not store secrets in these YAML files.** Keep local secrets in `.env`. For automated deployments, use the secret store provided by your CI/CD platform.

## 9. Build the Deployment Package

Build the development configuration:

```sh
uv run cdf build --config-yaml=config.dev.yaml
```

Toolkit reads the selected modules, substitutes environment variables, validates the configuration, and writes the final deployment files to `build/`.

Read the build output carefully. Fix validation errors before deploying, and review warnings and insights to catch configuration problems early.

To build another environment:

```sh
uv run cdf build --config-yaml=config.prod.yaml
```

> **Important:** `cdf deploy` uses the contents of `build/`. The most recent successful build therefore determines which environment and resources will be deployed.

## 10. Preview the Changes

A dry run shows what Toolkit plans to do without changing anything in CDF:

```sh
uv run cdf deploy --dry-run
```

Check:

- The target CDF project
- Resources that will be created
- Resources that will be updated
- Resources that might be removed or replaced
- Permission errors and validation warnings

If the target project or any planned change looks wrong, stop and correct the configuration before continuing.

## 11. Deploy to CDF

Once the dry-run output looks correct, deploy the built resources:

```sh
uv run cdf deploy
```

Toolkit sends the generated contents of `build/` to the authenticated CDF project.

For production environments, build the production configuration, dry-run it, and then deploy:

```sh
uv run cdf build --config-yaml=config.prod.yaml
uv run cdf deploy --dry-run
uv run cdf deploy
```

For production, use a reviewed CI/CD pipeline with client-credentials authentication whenever possible.

## 12. Understand the Project Structure

```text
<organization-folder>/
├── .env
├── .gitignore
├── .python-version
├── .venv/
├── cdf.toml
├── config.dev.yaml
├── config.test.yaml
├── config.prod.yaml
├── pyproject.toml
├── uv.lock
├── modules/
└── build/
```

- `.env` holds local credentials and must never be committed.
- `cdf.toml` contains the main Toolkit settings.
- `config.<env>.yaml` selects modules and variables for each environment.
- `modules/` contains the source configuration that your team maintains in Git.
- `build/` contains generated deployment artifacts.
- `pyproject.toml` and `uv.lock` define and lock the Python dependencies, and `.python-version` pins the Python version.
- `.venv/` is the local virtual environment that uv creates. Do not commit it.

## 13. Everyday Commands

```sh
# Check the installed Toolkit version
uv run cdf --version

# Show active Toolkit configuration
uv run cdf about

# Configure authentication
uv run cdf auth init

# Verify authentication and authorization
uv run cdf auth verify

# Initialize modules
uv run cdf modules init .

# Build an environment
uv run cdf build --config-yaml=config.dev.yaml

# Preview deployment
uv run cdf deploy --dry-run

# Deploy
uv run cdf deploy
```

## 14. Keep Toolkit Up to Date

When your team is ready to upgrade, update the locked Toolkit version and sync the environment:

```sh
uv lock --upgrade-package cognite-toolkit
uv sync
```

Check the installed version:

```sh
uv run cdf --version
```

Upgrade Toolkit modules when required:

```sh
uv run cdf modules upgrade
```

Review the release notes and all generated module changes, test the build, and commit the changes before deploying.

## 15. Notes for Windows Users

- Run the commands in Windows Terminal or PowerShell for the best interactive-menu experience.
- Install uv with `winget install --id=astral-sh.uv -e` or `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`.
- If `python` is unavailable, use `py` to check the installed Python versions, or let uv install one with `uv python install 3.11`.
- Use `uv run cdf ...`; you do not need to activate the virtual environment manually.
- If your organization blocks browser-based interactive authentication, try `device_code`.
- Paths containing forward slashes generally work in PowerShell.
- Install Git for Windows if the `git` command is unavailable. It also provides Git Bash.

## References

- [Setting up the Cognite Toolkit locally](https://docs.cognite.com/cdf/deploy/cdf_toolkit/guides/setup)
- [Authentication and authorization](https://docs.cognite.com/cdf/deploy/cdf_toolkit/guides/auth)
- [Configure, build, and deploy modules](https://docs.cognite.com/cdf/deploy/cdf_toolkit/guides/usage)
- [Cognite Toolkit configuration reference](https://docs.cognite.com/cdf/deploy/cdf_toolkit/api/index)
- [uv documentation](https://docs.astral.sh/uv/)
