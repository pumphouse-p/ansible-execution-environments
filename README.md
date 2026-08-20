![ee-main build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-main.yml/badge.svg)
![ee-iaas build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-iaas.yml/badge.svg)
![ee-servicenow build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-servicenow.yml/badge.svg)
![ee-terraform build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-terraform.yml/badge.svg)
![ee-controller build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-controller.yml/badge.svg)
![ee-containers build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-containers.yml/badge.svg)
![ee-de build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-de.yml/badge.svg)
![ee-windows build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-windows.yml/badge.svg)
![ee-ci build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-ci.yml/badge.svg)
![ee-ai build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-ai.yml/badge.svg)
![de-dt build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/de-dt.yml/badge.svg)
![de-kentik build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/de-kentik.yml/badge.svg)
![de-snow build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/de-snow.yml/badge.svg)

# ansible-execution-environments

This repo automatically builds and publishes container images for Ansible **Execution Environments** (EEs) and **Decision Environments** (DEs) using GitHub Actions. Each image is pushed to [quay.io](https://quay.io) and is ready to use with Ansible Automation Platform (AAP), `ansible-navigator`, or `ansible-runner`.

## Background: What Are Execution Environments and Decision Environments?

**Execution Environments (EEs)** are container images that bundle everything needed to run Ansible automation: a specific version of `ansible-core`, Python dependencies, system packages, and Ansible collections. Instead of installing all of these things on the machine where you run Ansible, you package them into a container image once and use that image everywhere. This eliminates "works on my machine" problems and makes your automation portable and reproducible.

**Decision Environments (DEs)** are the same concept, but for Event-Driven Ansible (EDA). They package the dependencies needed by EDA rulebooks -- things like event source plugins from collections such as `dynatrace.event_driven_ansible` or `servicenow.itsm`.

Both are built with a tool called [ansible-builder](https://ansible.readthedocs.io/projects/builder/en/latest/), which reads a definition file (`execution-environment.yml`) and produces a container image.

## Repository Structure

```
.
├── .github/workflows/          # GitHub Actions workflow files
│   ├── build-and-push.yml      # Reusable workflow (the engine that builds everything)
│   ├── ee-main.yml             # Caller workflow for ee-main
│   ├── ee-ci.yml               # Caller workflow for ee-ci
│   └── ...                     # One caller workflow per EE/DE
├── ansible.cfg                 # Galaxy server config (used during builds)
├── requirements.txt            # Pins ansible-builder version
├── ee-main/                    # EE directory
│   └── execution-environment.yml
├── ee-ci/
│   └── execution-environment.yml
├── de-dt/                      # DE directory
│   └── execution-environment.yml
└── ...                         # One directory per EE/DE
```

Each EE or DE has its own directory at the root of the repo (e.g., `ee-main/`, `de-dt/`). Inside that directory, the only file you need to create and maintain is `execution-environment.yml` -- this is the definition file that tells `ansible-builder` what to put in the image.

## How the `execution-environment.yml` File Works

This is the file you edit when you want to change what goes into an image. Here is a simplified example:

```yaml
---
version: 3

images:
  base_image:
    name: "registry.redhat.io/ansible-automation-platform-26/ee-minimal-rhel9:1.0.0"

dependencies:
  system:
    - git [platform:rpm]
  galaxy:
    collections:
      - name: amazon.aws
        version: 10.1.2
      - name: community.general
        version: 11.4.0
    roles:
      - name: geerlingguy.certbot
  python:
    - boto3
    - botocore

additional_build_steps:
  prepend_galaxy:
    - ENV ANSIBLE_GALAXY_SERVER_LIST=automation_hub,galaxy
    - ENV ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_URL=https://console.redhat.com/api/automation-hub/content/published/
    - ENV ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_AUTH_URL=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
    - ENV ANSIBLE_GALAXY_SERVER_GALAXY_URL=https://galaxy.ansible.com
    - ARG ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN

options:
  package_manager_path: /usr/bin/microdnf
```

Here is what each section does:

| Section | Purpose |
|---|---|
| `version` | The EE definition file format version. All definitions in this repo use version `3`. |
| `images.base_image` | The base container image to build on top of. EEs use `ee-minimal-rhel9` or `ee-supported-rhel9` from the AAP 2.6 registry. DEs use `de-minimal-rhel9`. |
| `dependencies.system` | RPM packages to install into the image (e.g., `git`, `gcc`, `krb5-devel`). |
| `dependencies.galaxy` | Ansible collections and roles to install. You can pin specific versions or leave them unpinned. |
| `dependencies.python` | Python packages (installed via `pip`) that your collections or playbooks need at runtime. |
| `additional_build_steps` | Raw Dockerfile/Containerfile instructions injected at specific points during the build. Commonly used to configure Galaxy server authentication so `ansible-builder` can download collections from Automation Hub. |
| `options.package_manager_path` | Tells `ansible-builder` to use `microdnf` (the minimal package manager on RHEL UBI images) instead of `dnf`. |

The `additional_build_steps.prepend_galaxy` section deserves extra attention. It sets environment variables inside the container during the build so that `ansible-galaxy` knows where to download collections from. The `ARG` lines declare build arguments that get passed in at build time (this is how the Automation Hub token is injected without hardcoding it in the file).

## How the GitHub Actions CI/CD Works

The build system uses two layers of GitHub Actions workflows: a **reusable workflow** that does the actual building, and individual **caller workflows** (one per EE/DE) that trigger it.

### The Reusable Workflow: `build-and-push.yml`

This is the core engine. It lives at `.github/workflows/build-and-push.yml` and does the following steps in order:

1. **Checks out the repo** so it has access to the EE definition files and `ansible.cfg`.
2. **Installs `ansible-builder`** by running `pip3 install -r requirements.txt` (which installs `ansible-builder==3.1.0`).
3. **Logs in to quay.io** using `redhat-actions/podman-login` so it can push the finished image.
4. **Logs in to registry.redhat.io** using `redhat-actions/podman-login` so it can pull the Red Hat base images (like `ee-minimal-rhel9`).
5. **Injects the Automation Hub token** into `ansible.cfg` by replacing the placeholder text `AUTOMATION_HUB_TOKEN_VALUE` with the real token from GitHub Secrets.
6. **Runs `ansible-builder build`** inside the EE's directory. This reads the `execution-environment.yml` file, generates a `Containerfile`, and builds the container image. The Automation Hub token is also passed as a build argument so it is available inside `additional_build_steps`.
7. **Pushes the built image** to `quay.io/<username>/<ee-name>:latest` using `redhat-actions/push-to-registry`.

This workflow accepts **inputs** (parameters) from the caller workflow:

| Input | Description |
|---|---|
| `EE_DIR_NAME` | The directory to build (e.g., `ee-main`, `de-dt`). |
| `EE_IMAGE_TAG` | The tag for the image (defaults to `latest`). |
| `REGISTRY_URL` | The container registry to push to (defaults to `quay.io`). |
| `REGISTRY_USERNAME` | The username on the registry (e.g., `deparris`). |

It also accepts **secrets** (sensitive values stored in GitHub):

| Secret | Description |
|---|---|
| `REDHAT_USERNAME` | Username for `registry.redhat.io` (to pull base images). |
| `REDHAT_PASSWORD` | Password for `registry.redhat.io`. |
| `REGISTRY_PASSWORD` | Password for `quay.io` (to push finished images). |
| `AH_TOKEN` | API token for Red Hat Automation Hub (to download certified/validated collections). |

### The Caller Workflows (One Per EE/DE)

Each execution environment has its own workflow file (e.g., `ee-main.yml`, `de-dt.yml`). These are small files that just call the reusable workflow with the right parameters. Here is a typical example:

```yaml
name: Main EE build

on:
  push:
    branches:
      - main
    paths:
      - 'ee-main/**'

  workflow_dispatch:

jobs:
  call-deploy-workflow:
    uses: pumphouse-p/ansible-execution-environments/.github/workflows/build-and-push.yml@main
    with:
      EE_DIR_NAME: ee-main
      EE_IMAGE_TAG: latest
      REGISTRY_URL: quay.io
      REGISTRY_USERNAME: deparris
    secrets: inherit
```

This workflow triggers a rebuild in two situations:

- **On push to `main`** -- but only when files inside the `ee-main/` directory have changed. This means editing `ee-main/execution-environment.yml` and pushing to `main` will automatically rebuild and publish the `ee-main` image.
- **On `workflow_dispatch`** -- this lets you manually trigger a rebuild from the GitHub Actions UI at any time (go to the Actions tab, select the workflow, and click "Run workflow").

Some workflows also have a **scheduled trigger** that rebuilds the image weekly (every Sunday at 01:30 UTC). This keeps the image up to date with the latest base image patches even when nothing in the definition has changed. The following workflows have this schedule: `ee-ci`, `ee-de`, `de-dt`, `de-kentik`, `de-snow`.

The `secrets: inherit` line passes all of the repository's GitHub Secrets down to the reusable workflow automatically.

## Available Execution Environments and Decision Environments

### Execution Environments (EEs)

| Name | Purpose | Key Collections |
|---|---|---|
| `ee-main` | General-purpose AWS and ServiceNow automation | `amazon.aws`, `community.aws`, `servicenow.itsm`, `community.general`, `containers.podman` |
| `ee-ci` | CI/CD pipeline automation | `containers.podman`, `community.general` + Python: `docker`, `python-jenkins` |
| `ee-containers` | Kubernetes, OpenShift, and container management | `kubernetes.core`, `redhat.openshift`, `redhat.openshift_virtualization`, `containers.podman` |
| `ee-controller` | AAP Config-as-Code (managing Controller, Hub, EDA, and Platform) | `ansible.controller`, `ansible.eda`, `ansible.hub`, `ansible.platform`, `infra.aap_configuration` |
| `ee-iaas` | Multi-cloud IaaS (AWS, Azure, GCP, OpenStack, VMware, Proxmox) | `amazon.aws`, `azure.azcollection`, `google.cloud`, `openstack.cloud`, `community.vmware`, `community.proxmox` |
| `ee-servicenow` | ServiceNow integration with AWS and Azure | `servicenow.itsm`, `amazon.aws`, `azure.azcollection` |
| `ee-terraform` | Terraform/OpenTofu and multi-cloud provisioning | `cloud.terraform`, `hashicorp.terraform`, `amazon.aws`, `google.cloud` + Terraform CLI and Azure CLI |
| `ee-windows` | Windows host management | `ansible.windows`, `community.windows`, `microsoft.ad`, `microsoft.iis`, `microsoft.sql`, `chocolatey.chocolatey` |
| `ee-ai` | AI/ML automation with Neubird | `neubird.aap` |
| `ee-de` | Decision environment with Dynatrace and ServiceNow | `dynatrace.event_driven_ansible`, `servicenow.itsm` |

### Decision Environments (DEs)

| Name | Purpose | Key Collections |
|---|---|---|
| `de-dt` | Dynatrace event-driven automation | `ansible.eda`, `dynatrace.event_driven_ansible` |
| `de-kentik` | Kentik network observability event-driven automation | `ansible.eda`, `kentik.ansible_eda` |
| `de-snow` | ServiceNow event-driven automation | `ansible.eda`, `servicenow.itsm` |

## How to Add a New Execution Environment

Follow these steps to create a new EE and have it build automatically.

### 1. Create the EE directory and definition file

Create a new directory at the root of the repo with your EE name (use the `ee-` or `de-` prefix convention). Inside it, create an `execution-environment.yml` file:

```bash
mkdir ee-myapp
```

Create `ee-myapp/execution-environment.yml` with content like:

```yaml
---
version: 3

images:
  base_image:
    name: "registry.redhat.io/ansible-automation-platform-26/ee-minimal-rhel9:1.0.0"

dependencies:
  galaxy:
    collections:
      - name: community.general
        version: 11.4.0
  python:
    - requests
  system:
    - git [platform:rpm]

additional_build_steps:
  prepend_galaxy:
    - ENV ANSIBLE_GALAXY_SERVER_LIST=automation_hub,galaxy
    - ENV ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_URL=https://console.redhat.com/api/automation-hub/content/published/
    - ENV ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_AUTH_URL=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
    - ENV ANSIBLE_GALAXY_SERVER_GALAXY_URL=https://galaxy.ansible.com
    - ARG ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN

options:
  package_manager_path: /usr/bin/microdnf
```

If your collections only come from Ansible Galaxy (not Automation Hub), you can omit the `additional_build_steps` section entirely.

### 2. Create a caller workflow

Create `.github/workflows/ee-myapp.yml`:

```yaml
name: MyApp EE build

on:
  push:
    branches:
      - main
    paths:
      - 'ee-myapp/**'

  workflow_dispatch:

jobs:
  call-deploy-workflow:
    uses: pumphouse-p/ansible-execution-environments/.github/workflows/build-and-push.yml@main
    with:
      EE_DIR_NAME: ee-myapp
      EE_IMAGE_TAG: latest
      REGISTRY_URL: quay.io
      REGISTRY_USERNAME: deparris
    secrets: inherit
```

### 3. Push to `main`

Commit both files and push to the `main` branch. The workflow will detect the change in `ee-myapp/**`, call the reusable workflow, build the image, and push it to `quay.io/deparris/ee-myapp:latest`.

### 4. (Optional) Add a build badge to the README

Add a badge to the top of this file:

```markdown
![ee-myapp build](https://github.com/pumphouse-p/ansible-execution-environments/actions/workflows/ee-myapp.yml/badge.svg)
```

## How to Modify an Existing Execution Environment

To change what is inside an existing image (add a collection, update a version, add a Python package, etc.):

1. Edit the `execution-environment.yml` file in the corresponding directory.
2. Commit and push to `main`.
3. The GitHub Action will detect the change and automatically rebuild and push the updated image.

You can also trigger a rebuild manually from the GitHub Actions UI without making any code changes (useful for picking up base image updates).

## Required GitHub Secrets

For the workflows to run, the following secrets must be configured in the GitHub repository settings (Settings > Secrets and variables > Actions):

| Secret | What It Is | Where to Get It |
|---|---|---|
| `REDHAT_USERNAME` | Your Red Hat account username | [Red Hat Login](https://sso.redhat.com) -- the same credentials you use for the Red Hat Customer Portal. |
| `REDHAT_PASSWORD` | Your Red Hat account password | Same as above. |
| `REGISTRY_PASSWORD` | Your quay.io password or token | [quay.io](https://quay.io) -- go to Account Settings > Generate Encrypted Password, or create a robot account. |
| `AH_TOKEN` | Automation Hub API token | [console.redhat.com](https://console.redhat.com/ansible/automation-hub/token) -- go to Automation Hub > Connect to Hub and copy the offline token. |

The workflows also use a GitHub Actions **environment** called `deploy`. You need to create this environment in your repository settings (Settings > Environments > New environment) and name it `deploy`.

## Building Locally

You can build any EE locally for testing before pushing changes. You need Python 3 and `podman` installed.

```bash
# Install ansible-builder
pip3 install -r requirements.txt

# Log in to the Red Hat registry (needed for the base images)
podman login registry.redhat.io

# Build an EE (from the repo root)
cd ee-main
ansible-builder build -t ee-main:latest
```

If the EE uses collections from Automation Hub, you will also need to either:
- Set the `ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN` environment variable, or
- Edit `ansible.cfg` to replace `AUTOMATION_HUB_TOKEN_VALUE` with your real token (do not commit this change).

## Key Files

| File | Purpose |
|---|---|
| `requirements.txt` | Pins the version of `ansible-builder` used in CI (currently `3.1.0`). |
| `ansible.cfg` | Configures Galaxy server endpoints for downloading collections. Contains a `AUTOMATION_HUB_TOKEN_VALUE` placeholder that CI replaces with the real token at build time. |
| `.github/workflows/build-and-push.yml` | The reusable workflow that all caller workflows invoke. This is where the actual build logic lives. |
| `.github/workflows/ee-*.yml` / `de-*.yml` | Caller workflows -- one per EE/DE. Each one sets the directory name and triggers the reusable workflow. |
| `*/execution-environment.yml` | The EE/DE definition file. This is the source of truth for what goes into each image. |
