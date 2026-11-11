# Customer deployment: private Azure DevOps Markdown POC

## What this profile delivers

Use the reviewed customer platform revision and its
`samples/customer-poc/azd-env.example.json` and
`ingestion-state.parameters.example.json`. These files are versioned with the
platform handover; do not assume upstream `main` contains fork-specific changes.
The deployment uses private classic Container Apps UI/orchestrator, Foundry IQ
`searchIndex`, externally published Markdown chunks, and separate private Blob
control state. It does not need Content Understanding, a Search Blob indexer,
conversation uploads, the bundled ingestion app or hosted Agent Service.

The platform and runtime components are separate repositories. Obtain the UI
and orchestrator revisions pinned by the delivered `manifest.json` and
`app-definition.json`, plus a reviewed immutable revision of the independent
Markdown worker. Customer access to that worker repository, template checkout
and source repositories must be arranged separately. Never copy a developer's
dirty component checkout or registry credentials as a delivery mechanism.

The customer branch now selects the tested runtime fixes from public forks:

| Component repository | Source branch | Required full commit |
| --- | --- | --- |
| `AsmaeFILALI/agent-app-ui` | `feature/customer-build-feeds` | `132f70bf862a482433faf4fccbdbf09cf8f1e12a` |
| `AsmaeFILALI/agent-app-orchestrator` | `feature/citation-guard` | `72204060a055ed6081d0cc4d08d221275e907d1f` |

These are reviewed source candidates, not newly published upstream releases.
Component tag fields are intentionally empty. The existing deployer uses the
branch to fetch source and verifies its full commit before building; moving a
branch to a different commit fails the pin check. The matching
`app-definition.json` commits are updated together with the manifest.
Use that delivered platform revision, not the fork's default component branch.
Both clean Linux/amd64 image builds and offline checks passed, with a
credential-free HTTPS package mirror required on the validation workstation.
No registry image or Azure application update follows from publishing source.
An image tag containing a short commit ID does not prove a clean build.

## 1. Prerequisites and policy

Use Azure CLI, Azure Developer CLI, Git, PowerShell 7 or Bash with jq, and
Python 3.12. Local image builds require Docker with Linux/amd64 support.
The worker requires its reviewed uv version, Python 3.12 and `uv.lock`.
Allow approved package/container registry and tokenizer-download egress during
dependency preparation. The platform hooks also bootstrap pip over HTTPS;
an entirely offline build path is not provided by this profile.

Before any writes, have administrators approve:

- Customer tenant/subscription and deployment principal **object ID**. An
  application/client ID is not a service-principal object ID.
- Resource deployment permissions and scoped role-assignment authority;
  configuration operations additionally need the relevant service data roles.
- Required provider registration, model/version/SKU capacity, Search semantic
  capabilities and subscription policy. ARM validation does not guarantee capacity.
- Ongoing VPN gateway, DNS resolver, NAT, private endpoint, database, model,
  Search and application costs.
- The default primary/Search region Canada East and application Cosmos DB
  Canada Central, or a separately reviewed region selection.
- Model processing policy: the profile retains the configured GlobalStandard
  chat SKU. Canadian resource placement is **not** Canada-only inference.
- Preview API use. Embedding inference, Search schema administration, knowledge
  retrieval and ARM APIs have different versions; do not interchange them.

The current address profile is:

| Purpose | Range |
| --- | --- |
| VNet | `10.77.0.0/21` |
| Agent / Container Apps | `10.77.0.0/24` / `10.77.1.0/24` |
| Private endpoints | `10.77.2.0/26` |
| Reserved Bastion / Firewall / Gateway | `10.77.2.64/26` / `10.77.2.128/26` / `10.77.2.192/26` |
| App Gateway / jumpbox / build agents | `10.77.3.0/27` / `10.77.3.64/27` / `10.77.3.96/27` |
| DNS resolver | `10.77.3.128/28` |
| VPN clients | `172.27.240.0/24` |

Reserved subnets do not imply those optional services are deployed. Keep these
ranges for this POC; check that customer connected networks do not overlap.
If there is a conflict, stop and request a separate network design change.

## 2. Prepare a new environment

Clone the customer-accessible platform repository and check out the exact
handover revision. Do not copy someone else's `.azure` directory. From its root:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r .\config\requirements.txt
az login --tenant <customer-tenant>
az account set --subscription <customer-subscription>
azd auth login --tenant-id <customer-tenant>
```

In Bash, activate with `source .venv/bin/activate` and use the equivalent native
paths. Confirm both tools target the customer context. Clear unrelated global
Azure CLI location filters; resource discovery must not hide services located
outside the main region.

Copy the JSON profile to an ignored location such as `.local/customer.json`.
Replace every required placeholder. `inputs` contains nonsecret operator
choices; `bootstrap` and `connected` are mutually exclusive phase overlays.
Do not add generated service names, endpoints, secrets, migration app names or
old resource IDs to this file.

```powershell
$profile = Get-Content -Raw .\.local\customer.json | ConvertFrom-Json
if (($profile | ConvertTo-Json -Depth 10) -match '<[^>]+>') {
    throw 'Replace every profile placeholder before creating an environment.'
}
azd env new $profile.inputs.AZURE_ENV_NAME --no-prompt
if ($LASTEXITCODE -ne 0) { throw 'Environment initialization failed.' }
foreach ($section in @($profile.inputs, $profile.bootstrap)) {
    foreach ($entry in $section.PSObject.Properties) {
        # Empty optional inputs use the Bicep fallback in this fresh environment.
        if ($entry.Value -eq '') { continue }
        azd env set $entry.Name $entry.Value --no-prompt
        if ($LASTEXITCODE -ne 0) { throw "Cannot set $($entry.Name)." }
    }
}
```

Fresh defaults are hosted orchestration/native Blob/standard schema, so the
explicit profile is essential. It selects `classic`, `maf_lite`, private
`searchIndex`, `ado_markdown_v1`, equal `SEARCH_RAG_INDEX_NAME` and
`FOUNDRY_IQ_SEARCH_INDEX_NAME`, and `semantic-config`. It disables optional
ingestion/uploads, hosted backing services, panel, jumpbox/Bastion/VM/firewall
and ACR Tasks pool while retaining NAT egress and local Docker builds.

`COSMOS_ACCOUNT_NAME` is empty for generated globally unique names. For an
**existing** environment, first set it to the actual selected account before
adopting the portable parameters, and review a no-replacement preview. The
`DATABASE_ACCOUNT_NAME` output is not the new input. Do not change database
identity as a retry shortcut.

## 3. Provision, connect, then complete service configuration

Follow [Deploy infra only](deploy-infra-only.md), including composition and
regional preflight. The example's bootstrap overlay explicitly defers
post-provision data-plane setup because the VPN does not exist yet:

```powershell
azd provision --no-prompt
```

Successful foundation provisioning with deferral is **not application readiness**.
No app image is built by that step. Keep public access disabled.

Read `pointToSiteVpnDeployment` outputs from the configured network resource
group, generate/import the gateway's Azure VPN Client profile, authorize Entra
sign-in and set its custom DNS to the deployed resolver address. Do not copy
our profile, resolver IP or private endpoint IPs. The template does not import
the client profile or perform interactive sign-in.

Follow [Private access](private-access.md) to verify routing and normal OS DNS
for App Configuration, Key Vault, Search, OpenAI and ACR. All service names must
resolve privately and validate TLS. DNS records alone do not configure a laptop.

After connecting, apply the connected overlay and run the matching hook:

```powershell
foreach ($entry in $profile.connected.PSObject.Properties) {
    azd env set $entry.Name $entry.Value --no-prompt
    if ($LASTEXITCODE -ne 0) { throw "Cannot set $($entry.Name)." }
}
pwsh -NoProfile -File .\scripts\postProvision.ps1
```

On Bash hosts use `bash ./scripts/postProvision.sh`. The hook configures roles,
governance/Key Vault, App Configuration, model/service references, the Markdown
index and knowledge objects. It can reconcile platform-owned objects: do not
run it against unrelated serving resources as a generic repair step.

Verify `AGENT_STRATEGY=maf_lite` and `CHAT_BACKEND=orchestrator` under `agent-lz`,
the exact index/model/vectorizer, `semantic-config`, and source-data projection
including `url`, `filepath`, `content`, `heading_path` and provenance. Environment
variables only affect runtime when explicitly published or read by the component.

## 4. Configure authentication and deploy applications

Follow [Application access](application-auth.md) for a customer-owned
single-tenant Entra web app, consent and allowed users. The canonical UI callback
is `<UI-HTTPS-origin>/auth/oauth/azure-ad/callback`. Configure OAuth client/tenant,
scopes, `CHAINLIT_URL` and persistent session-signing secret. Credentials belong
in Key Vault, referenced through the approved configuration path, not this JSON
profile. Keep `ALLOW_ANONYMOUS=false`; VPN authentication is not application login.

From a connected build host:

```powershell
azd deploy --no-prompt
```

The parent deployment fetches the pinned components and checks tracked and
untracked source cleanliness before copying the environment or building.
Dirty component source fails without resetting files. Use a separate clean
checkout or review/commit changes first. Do not work around the guard by hiding
source edits with ignore flags. The selected fork commits support credential-free
HTTPS `PIP_INDEX_URL` overrides for local builds; the orchestrator also supports
`NPM_CONFIG_REGISTRY`. Configure a customer-approved source if direct package
artifact downloads are blocked. Authenticated feed customization needs a separate
secure build-secret review; do not put credentials in these build arguments.

Verify both active revisions, recorded image digests, authenticated UI access,
orchestrator connectivity and a known-answer query. Do not infer success from
Container Apps provisioning state alone.

## 5. Deploy the independent control-state store

Create/select the customer WIF publishing service principal with the
administrator before supplying `writerPrincipalId`. Copy and fill
`samples/customer-poc/ingestion-state.parameters.example.json` in an ignored
location. Use the actual workload private endpoint subnet and linked Blob
private DNS zone ARM IDs. Defaults generate new account/RG names.

After preview review and explicit write approval:

```powershell
az deployment sub validate --location canadaeast --subscription <subscription> `
  --template-file .\infra\ingestion-state.bicep --parameters .\.local\state.parameters.json
az deployment sub what-if --location canadaeast --subscription <subscription> `
  --template-file .\infra\ingestion-state.bicep --parameters .\.local\state.parameters.json
az deployment sub create --name <state-deployment> --location canadaeast `
  --subscription <subscription> --template-file .\infra\ingestion-state.bicep `
  --parameters .\.local\state.parameters.json
```

This account is private LRS/Hot with Shared Key/anonymous access disabled,
versioning and 14-day Blob/container soft delete. It is not Markdown staging or
a Search indexer source. The worker creates publisher/checkpoint/run blobs;
infrastructure leaves the container empty. Verify actual flags, Approved
endpoint, DNS record, exact writer role and private authenticated access.

## 6. Export a reviewed worker handoff

Capture **full successful deployment snapshots**, not raw environment dumps:

```powershell
az deployment group show --subscription <subscription> --resource-group <app-rg> `
  --name <main-deployment> --output json > .\.local\main-deployment.json
az deployment sub show --subscription <subscription> --name <state-deployment> `
  --output json > .\.local\state-deployment.json
```

Render/read-back the customer index through `python -m config.search.markdown_index`
using explicit `--search-endpoint`, `--index-name`, `--embedding-endpoint`,
`--embedding-deployment` and `--schema-output`. Without `--apply` this command
does not create or update an index. Transfer the exact full schema plus its
independently reviewed SHA-256; a consumer requirements JSON is not a schema.

```powershell
python -m config.deployment.ingestion_handoff `
  --platform-deployment .\.local\main-deployment.json `
  --state-deployment .\.local\state-deployment.json `
  --platform-schema .\.local\customer-index.schema.json `
  --schema-sha256 <owner-reviewed-sha256> --index-name <explicit-index> `
  --serving-status <serving-or-non-serving> `
  --search-schema-api-version 2025-05-01-preview `
  --output .\.local\ingestion-handoff.json
```

This exporter is offline and allowlists bindings. It verifies successful
snapshots, environment/subscription/account consistency, embedding settings
and exact pinned schema. It refuses to overwrite inputs or existing outputs.
It does not prove live access, change an index, create admission or declare a
test target safe. `serving-status` is an explicit owner assertion, not an
inference from a name containing `test`. A serving target must match the main
deployment's recorded binding; verify current Azure bindings separately.

The `publishing` object maps to the worker publishing TOML, except that local
schema path, WIF client/service-connection IDs and admission remain owner inputs.
Use `embedding` for source model/deployment/dimensions. The state writer may be a
development user; ensure the actual publishing principal has its own grants.
Treat deployment snapshots as private working evidence; only the allowlisted
handoff is intended for transfer. Review it under the customer's data policy.

## 7. Onboard the independent worker

Follow the delivered worker's `docs/deployment.md` and `docs/operations.md`.
Pin its template repository tag and full commit. Prepare source repository ID,
full ref, immutable commit and include/exclude rules; platform schema/hash;
owner admission; protected environment; and WIF connection authorized only
to the ingestion pipeline.

The worker needs selected-index document access, separate read-only schema
permission, OpenAI inference and exact container-scoped Blob permission.
Search's query-vectorizer identity also needs the platform-managed embedding
access. Repository Build Service checkout permissions are separate.

Private publication requires an approved VNet-connected agent with private DNS.
A laptop VPN does not connect Microsoft-hosted agents. Start with
`enablePublishing=false`; validate the exact source offline. Only after owner
approval run first publication, verify checkpoint/documents, repeat unchanged
and verify no new embedding/document work. Rehearse interrupted-run recovery.
Never use an unsigned test waiver against an index already serving users.

## 8. Local development

Use separate clean component checkouts at the delivered pins, independent
Python 3.12 virtual environments and the component's documented dependencies.
On the connected workstation, sign into the customer tenant and arrange scoped
developer access to configuration, Key Vault and the required services. Managed
identity in Azure and local CLI credentials are separate authentication paths.

For the delivered upstream baseline, run from each component root:

```text
# UI: install requirements and the package as described in docs/python-development.md
uvicorn main:app --host 127.0.0.1 --port 8080
# Orchestrator: install its requirements in a separate environment
uvicorn main:app --app-dir src --host 127.0.0.1 --port 9000
```

Set `APP_CONFIG_ENDPOINT` explicitly for each process and follow component
instructions for runtime references. For local UI-to-orchestrator calls, review
the local URL/caller authentication settings rather than changing shared Azure
bindings. Configure the local OAuth callback/allowed origins and localhost
scheme according to the component's supported auth flow. Do not weaken the
deployed app or fabricate identity headers to make local chat work.

For worker-only development use `uv sync --locked --python 3.12 --group dev`,
prepare its verified tokenizer cache, then run `uv run --locked ado-ingest plan`
with the reviewed source TOML. `plan` is offline after dependencies/cache are
prepared. `sync`/`reconcile`/`rebuild` are publishing operations, not local
preview commands. Customer-specific authenticated startup remains an acceptance
check; these entrypoints alone do not establish end-to-end readiness.

## Acceptance, troubleshooting and recovery

- Check a clean second checkout reproduces profile composition and builds.
- Verify public-network-disabled resources, normal private DNS/TLS and scoped
  runtime/worker permissions; a 403 can be networking or authorization.
- Confirm generated names, no accidental old POC accounts, exact image digests,
  completed configuration and persisted runtime strategy.
- Validate full schema, identical ingestion/query embeddings and source citations.
- Validate no-change ingestion and interruption recovery with synthetic content.
- Preserve content/state on failed setup; do not delete/recreate accounts/indexes.
- Restore compatible image/config/schema bindings deliberately. Do not run full
  hooks as a destructive index repair or interpret Blob restore as a Search
  transaction rollback.

Azure DevOps permissions are not automatically copied into Search. Until an
approved permission contract and allow/deny tests pass, use synthetic or
explicitly admitted common-audience non-sensitive content only. Authentication,
private networking and a fluent answer do not establish document authorization.

Before customer delivery record the exact platform/component/worker commits,
image digests, documentation revision, customer schema/hash, nonsecret bindings,
tested commands and outstanding gates. Publication, a new paid validation
deployment and customer access changes require separate authorization.
