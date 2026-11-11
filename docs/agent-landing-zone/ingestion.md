# Ingestion and conversation uploads

The [customer Markdown POC runbook](azure-devops-markdown-poc.md) connects
platform provisioning, the separate control-state store and the independent
worker. Its offline handoff exporter checks exact customer schema/model/bindings
but does not grant access, create source admission or authorize publication.

Foundry IQ provides the standard source-ingestion path. Conversation attachments
and custom processing are separate capabilities that can require optional
**Agent App Ingestion**.

This page explains those boundaries. Deployment steps are in the
[Deploy full stack runbook](deploy-full-stack.md); source and index inputs are
described in [Deployment settings](configuration.md#ingestion-application-selection).

<span id="choose-a-source-ingestion-path"></span>

## 1. Standard ingestion with Foundry IQ

The default source is Azure Blob Storage, with `FOUNDRY_IQ_PATTERN=azureBlob`
and container `documents`. Azure AI Search, which powers Foundry IQ, creates
and manages the indexing pipeline for this knowledge source. It extracts
content, splits it into smaller passages (**chunks**), generates **embeddings**
(vectors used for similarity search) and populates an index used by the
knowledge base. **Agent App Ingestion is not required** for this path.
New Agent Application environments default to
`DEPLOY_INGESTION_CONTAINER_APP=false`.

Source documents belong in the configured Blob container and optional folder,
not in a chat attachment. Search's ingestion identity needs access to storage
and the configured extraction and embedding services. The default permission
metadata is `["rbacScope"]`; ingestion permissions and the querying user's
authorization remain separate requirements.
Microsoft's [Blob knowledge-source guide](https://learn.microsoft.com/azure/search/agentic-knowledge-source-how-to-blob)
describes the managed pipeline and service requirements.

Successful ingestion does not establish protected retrieval in the hosted
application. The shipped hosted entrypoint supplies empty user context to the
retrieval strategy, so the connector cannot forward an on-behalf-of (OBO) token.
This limitation affects protected Blob retrieval as well as the sources below.

<span id="sharepoint-is-a-separate-integration"></span>

Foundry also supports native SharePoint options, including previews:

| Native option | What happens to the content |
| --- | --- |
| Foundry IQ `indexedSharePoint` source | A service-managed pipeline copies and processes SharePoint content into a Search index. No Agent App Ingestion is required by the service. |
| Foundry IQ `remoteSharePoint` source | The knowledge base queries Microsoft 365 at retrieval time. It does not copy the SharePoint corpus into a landing-zone index. Retrieved text still reaches the agent and model. |
| Foundry Agent Service SharePoint tool | The agent calls a native tool backed by Microsoft 365 retrieval, outside the knowledge-base source list. It does not require the custom ingestion application. |

These product options are not all wired into Agent Application. The indexed
SharePoint template needs a source-contract update; remote retrieval and native
tools need their own connection, consent and trusted delegated-user integration.
Native tool calls require the user's identity, not application-only access.
[SharePoint](sharepoint.md) explains the options and current integration gaps.
Preview status is a governance consideration, not a reason to assume custom
ingestion is required.

The accelerator's ingestion-free validation accepts native Blob or an explicitly
bound, independently populated Search index. A SharePoint-only replacement needs
source-specific integration and authorization validation; a product-supported
source is not automatically a supported deployment selection.

<span id="conversation-uploads-are-not-native-source-ingestion"></span>

## 2. Conversation file uploads

A chat attachment is processed for one conversation, not added to the shared
native Blob corpus. This path **requires Agent App Ingestion**, its reachable
endpoint and a compatible Search index. Alongside the default native Blob
source, the additional conversation `searchIndex` source needs
`FOUNDRY_IQ_CONVERSATION_UPLOAD_ENABLED=true`.
The flag is invalid without the ingestion app.

With `DEPLOY_INGESTION_CONTAINER_APP=false`, Agent App UI hides attachment
controls and rejects server uploads and client preflight before resolving an
ingestion endpoint or Dapr fallback. Setting it to `true` permits the upload
path, but does not make authenticated retrieval work automatically.

The UI's attachment controls require authenticated access. Anonymous Copilot
sessions cannot upload. The chat handler accepts PDFs, up to five files,
15 MiB per file and 25 MiB total. Hosted uploads require the UI's existing hosted
conversation identifier. A text turn establishes it in the non-continuity hosted
path; continuity-enabled chat does not populate that upload identifier.

The upload payload can carry the uploader's object ID as permission metadata.
That is not sufficient by itself: the index must enforce those permissions and
retrieval must preserve conversation scope and trusted user authorization.
**Conversation scope** uses the hosted backend's conversation namespace,
not the local UI thread ID. The same identifier must reach the indexed upload
and retrieval filter. Retrieval must match both `conversationId` and the trusted
uploader in `metadata_security_user_ids`; the identifier alone grants no access.

The default hosted entrypoint supplies no authenticated owner, so the native
Blob conversation-upload source is skipped. The delegated-token gap described
above also affects protected conversation retrieval. Enabling ingestion does not resolve it.
The [conversation PDF reference](multimodality.md#attach-a-pdf-to-a-conversation)
describes source bindings. Hosted use still needs the reviewed
[Foundry IQ identity integration](foundry-iq.md).

<span id="select-the-optional-application"></span>
<span id="select-agent-app-ingestion"></span>
<span id="upgrade-and-retirement-safety"></span>

## 3. Custom ingestion only when needed

Agent App Ingestion is useful when the managed source does not meet processing
requirements, or when conversation uploads or the application's custom
SharePoint jobs are required. Its format-specific processing performs chunking
and embedding generation and writes content to Search. An independent pipeline
can instead populate a compatible index exposed through a Foundry IQ
`searchIndex` source.

The optional application's SharePoint connector uses Microsoft Graph application
credentials and indexing/purge jobs. It copies content and permission metadata
into Search; permission synchronization and query-time enforcement still need
validation. It is separate from native SharePoint knowledge sources and the
native Foundry tool. Its `SHAREPOINT_*` credentials are not the
`SHAREPOINT_REMOTE_*` retrieval settings. Secrets belong in Key Vault.

For the bundled Agent Application, `DEPLOY_INGESTION_CONTAINER_APP` selects
the optional component through the [Deploy full stack runbook](deploy-full-stack.md),
not a runtime-only edit. Existing environments retain their recorded selection;
custom application definitions retain their declared components. Selection does
not change retrieval bindings or document authorization.
Omitting the application does not delete deployed resources, indexed documents
or identity grants; retirement needs separate authorization.

### Platform-owned Markdown index deployment

An independent Azure DevOps Markdown worker owns document synchronization,
not the Search schema. The platform's `contracts/ado-markdown-index-consumer-v1.json`
pins the draft consumer requirements; it is **not** a deployable index body.
The authoritative RAG index definition is `config/search/rag-index.j2`, shared
with the normal Search setup through `search.j2`.

The opt-in Markdown rendering adds source/section/line/pipeline metadata,
explicit `parent_id` filtering and the `semantic-config` priorities:
title `title`, content `content`, keywords `heading_path`, then `category`.
It retains bundled platform fields, 3072-dimensional `contentVector`,
HNSW/cosine, and the Azure OpenAI query vectorizer using the same
`text-embedding-3-large` model as ingestion.

For repeatable provisioning of a serving Markdown index, explicitly select:

```powershell
azd env set SEARCH_RAG_SCHEMA ado_markdown_v1
azd env set SEARCH_RAG_INDEX_NAME kb-sections-ingestion-test-v1
azd env set FOUNDRY_IQ_SEARCH_INDEX_NAME kb-sections-ingestion-test-v1
azd env set RETRIEVAL_BACKEND foundry_iq
azd env set FOUNDRY_IQ_PATTERN searchIndex
azd env set FOUNDRY_IQ_KNOWLEDGE_SOURCE_KIND searchIndex
```

`SEARCH_RAG_SCHEMA` is a platform setup selector published under `agent-lz`,
not an application runtime strategy. Its default `standard` preserves existing
deployments. `ado_markdown_v1` requires an explicit index name and either
`ai_search` or Foundry IQ `searchIndex` retrieval; conflicting legacy index-name
overrides and native Blob selections fail validation.

Pre-provisioning publishes this selection through the existing generic
`additionalAppConfigurationSettings` parameter. Both post-provision hooks
publish it consistently. Search setup renders the same shared definition and
uses the standalone command's create-or-validate implementation **before**
knowledge-source/base reconciliation. A compatible existing Markdown index is
not written, additively updated, dropped or recreated; its documents are not
modified. Missing indexes are created conditionally and read-back validated.
Schema/vectorizer/semantic mismatches abort before other index writes and
knowledge cleanup. Other standard indexes keep their existing behavior.

Persist the knowledge-source/base names and embedding binding for the target
environment as well. A portal-only change is not durable deployment intent.
If you create the knowledge source yourself, set `sourceDataFields` to include at
least `url`, `filepath`, `title`, `content` and `heading_path`. Without a
projected `url`/`filepath` the retrieval response has no citation link, and the
model may fabricate one.
An index becomes a serving target once a knowledge source/base or application
references it, regardless of a `test` substring in its name. For incompatible
schema revisions, create a new generation and validate an approved cutover;
do not reset a populated index or switch the selector just to avoid a failure.

The internal render-only `MARKDOWN_INDEX` input remains for template reuse; it
is not an environment-variable substitute for `SEARCH_RAG_SCHEMA`. No new
application component version or Search indexer is required. Ingestion still
pushes documents directly; state Blob storage is not a Search data source.

Run the platform-owned **index-only command** after infrastructure and identity
provisioning. Use the repository's existing Python environment with
`config/requirements.txt` installed, a private-network-connected runner, and an
approved deployment identity with Search schema-management permissions.
The worker's separate document-writer identity need not manage schemas.

```powershell
# Read-only preflight; exports a deployable definition and its SHA-256 sidecar.
python -m config.search.markdown_index `
  --search-endpoint https://<search-service>.search.windows.net `
  --index-name kb-sections-ingestion-test-v1 `
  --embedding-endpoint https://<foundry-account>.openai.azure.com `
  --embedding-deployment text-embedding `
  --schema-output .\markdown-index.schema.json

# After reviewing the exact target and obtaining approval, run the same command
# with --apply. It conditionally creates only a missing named index.
```

The command defaults to Search API `2025-05-01-preview`, matching the platform
schema's permission annotations; this is independent of the worker's OpenAI
inference API version. A missing index is created with `If-None-Match: *` and
read back for validation. A compatible existing index is validated without a
write. Any incompatible field, vectorizer binding or semantic priority fails
explicitly; the command never adds fields, drops/recreates indexes, changes
documents, publishes App Configuration, or reconciles knowledge objects.
After a concurrent/ambiguous create failure, rerun read-only validation rather
than deleting the index.

Do not invoke the full post-provision hook just to create a test index: full
Search setup also reconciles knowledge sources and knowledge bases. The
standalone command reuses its conditional index API helpers without running
that broader flow. Keep the application index setting and Foundry IQ bindings
unchanged while testing a non-serving target.

Give the ingestion project the exported **platform schema and its exact hash**,
consumer contract version/hash, target endpoint/name and embedding binding.
Publishing must fail before writes if the target or pinned definition is
incompatible. Physical field count can exceed the 23 required consumer fields.
Provisioning proves schema compatibility only: citations, source synchronization,
model consistency, vectorizer execution and allowed/denied retrieval still need
their own tests. The draft's authorization/principal mapping is unresolved;
use only synthetic or explicitly approved non-sensitive content until settled.

### Independent worker control-state storage

An independent Azure DevOps Markdown worker can publish directly to Search
without staging Markdown in Blob or creating a Search indexer. Keep its
**control state** separate from the accelerator's document/upload storage.
The optional standalone `infra/ingestion-state.bicep` entry point creates:

- A dedicated ingestion resource group and StorageV2 Standard_LRS/Hot account.
- One private `ingestion-state` container.
- Blob versioning and 14-day Blob/container soft-delete retention, without an
  automatic expiration policy or container-wide WORM/legal hold.
- A Blob private endpoint using an existing endpoint subnet and Blob private
  DNS zone. It does not create another VNet, zone, resolver or VPN gateway.
- One explicit container-scoped Storage Blob Data Contributor assignment.
  Supply a worker's managed identity/WIF service principal for automation, or
  an explicitly approved user for development. It does not grant Search,
  UI or orchestrator identities access.

The account disables public network access, anonymous Blob access and Shared
Key, and requires HTTPS/TLS 1.2+. Retention, location, resource names and
container name are explicit parameters. LRS is suitable for the approved POC,
not a substitute for a reviewed regional recovery strategy.

Deploy this entry point **independently**, not through a full accelerator
reprovision or post-provision hook. Use Azure CLI with reviewed native ARM
parameters in a local `state.parameters.json` file. Include `environmentName`,
`location`, `privateEndpointSubnetResourceId`, `blobPrivateDnsZoneResourceId`,
`writerPrincipalId`, and `writerPrincipalType` (`ServicePrincipal` by default).
The subnet must be in the same region as the endpoint. Use optional
`ingestionResourceGroupName` and `storageAccountName` overrides to preserve
existing resource identities on subsequent deployments.

```powershell
az deployment sub validate --subscription <subscription-id> --location <region> `
  --template-file .\infra\ingestion-state.bicep --parameters .\state.parameters.json
az deployment sub what-if --subscription <subscription-id> --location <region> `
  --template-file .\infra\ingestion-state.bicep --parameters .\state.parameters.json
# Review and approve the additive resource plan before execution.
az deployment sub create --name ingestion-state --subscription <subscription-id> `
  --location <region> --template-file .\infra\ingestion-state.bicep `
  --parameters .\state.parameters.json
```

Outputs under `ingestionState` expose nonsecret account/Blob endpoint, container,
resource group, private endpoint and the writer assignment. They do not change
the platform's existing `STORAGE_ACCOUNT_NAME`, document containers, App
Configuration keys or retrieval bindings. Never supply storage keys or a SAS.

Bicep leaves the container empty. The worker creates virtual prefixes and JSON:

```text
ingestion-state/
  indexes/<physical-index-id>/
    publisher.json
    scopes/<source-id>/
      checkpoints/<manifest-id>.json
      runs/<manifest-id>.json
```

Use a renewable lease on the mutable publisher blob and conditional writes;
create run intents and successful checkpoints with create-only conditions.
Write intent before Search mutations, verify all expected document results,
then create a successful checkpoint and conditionally advance committed
pointers. A Blob lease does not fence Search writes or provide an atomic
Search/Blob transaction. Stop new writes after lease loss and reconcile partial
or ambiguous outcomes before committing. Do not select committed state solely
by the latest timestamp. Never infer deletions from failed/partial enumeration
or blindly age-delete referenced checkpoints and unresolved intents. Prefixes
are not independent RBAC boundaries.

The publishing runner needs **its own** private routing/DNS and Entra access;
a laptop VPN does not connect Microsoft-hosted pipeline agents. Before wiring
the worker, test authenticated conditional create/update and lease contention
with disposable blobs over the private endpoint. A successful ARM deployment
alone does not establish worker readiness or protected retrieval.

<span id="authorization-remains-a-design-requirement"></span>
<span id="related-references"></span>

For every path, acceptance includes a document that an allowed user can
retrieve with its citation and that a denied user cannot retrieve. Source
permissions, query-time enforcement and private-network connectivity require
separate validation. Private Blob ingestion is covered in
[Network isolation](network-isolation.md#private-native-blob-ingestion).
