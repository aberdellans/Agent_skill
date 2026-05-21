---
name: inveniam
description: Operate the Inveniam.io platform API as an AI agent — listing and managing deals and portfolios, walking data-room folders and documents, reading extracted data artifacts and taxonomies, running file-veracity checks, and driving per-deal workflows with tasks, checklists, RACI assignments, and comments. Use this skill for tasks involving the Inveniam platform — an Inveniam deal or data room, an Inveniam portfolio, Inveniam deal documents, workflow tasks, RACI, or file veracity — including cases that are clearly Inveniam-related even when the platform is not named explicitly. It covers the access options (an Inveniam MCP server or the V2 REST API directly), the resource model, the task-status enum and transition rules, data-room write limitations, and the preview-and-confirm rule for destructive operations.
---

# Inveniam platform

This skill helps an AI agent operate the **Inveniam.io platform** — investment deals and their supporting data rooms. The platform's capabilities are exposed through a V2 REST API, and everything below — the resource model, the operations, the idiosyncrasies, the recipes — holds no matter how you reach that API. It captures what is *not* obvious from a tool list or an endpoint name, so you spend calls correctly instead of rediscovering quirks by trial and error.

## Reaching the platform — which interface to use

There are three ways an agent can drive the Inveniam platform. Prefer them in this order:

1. **The official Inveniam MCP server — use this when it is available.** Inveniam is building a first-party, hosted MCP server that wraps the platform API. As of May 2026 it is in active development and partial rollout: authentication and the core deal/organization tooling are well along, workflow and settings coverage is still in progress, and it is not yet generally available. When it ships, prefer it over anything local — it is first-party (kept in sync as the API evolves), hosted with proper OAuth, and, being a remote server, usable from any agent host that supports remote MCP connectors rather than only a local install. Confirm its current availability and connection details with Inveniam.

2. **The Inveniam V2 REST API, directly.** The platform API is documented, and an agent that can make authenticated HTTP calls can use it with no MCP server at all. Auth is a two-step exchange: trade the API key plus a long-lived token for a short-lived JWT, then send that JWT as a bearer token. This path is always available, and it is the ground truth that every MCP server merely wraps.

3. **A local stopgap MCP server (e.g. `inveniam-mcp`).** A self-hosted wrapper that re-exposes the V2 API as MCP tools. It works today and is fine for a local-agent host that can launch a local process, but treat it as a **stopgap until the official MCP ships**: it is local-only (a cloud-hosted agent cannot reach it), it must be maintained by hand as the API changes, and it will be superseded by — and is inferior to — the first-party server. Don't build long-term tooling around it. To troubleshoot one: it is registered in the host's MCP configuration; if its tools are missing, or every call returns HTTP 401, the server process or its credentials file is what to check — not the tool arguments.

**A note on tool names.** The names used throughout this skill (`list_deals`, `get_deal`, `change_task_status`, …) are operation-level names typical of a wrapper. The official MCP will expose equivalent operations under its own names, and against the REST API they map to `/v2/...` endpoints. What is stable — and what this skill is really about — is the *operations and the platform's behavior*, not the surface you call them through.

## Resource model

Inveniam resources nest like this:

```
Portfolio  →  Deal  →  Folder  →  File (document)
```

Each **Deal** also carries:

- **Data artifacts** — structured data the platform's "datalab" pipeline extracted from documents.
- **Document taxonomies** — classification metadata assigned to each document.
- **Workflows** — one or more workflow instances, each with **tasks**; tasks have **checklists**, **RACI assignments** (Responsible / Accountable / Consulted / Informed), and **comments**.

Almost every tool is scoped by a `dealId` (a UUID), so obtain that first.

## Where to start

Begin with `list_deals` or `list_portfolios`, then drill down. Never guess UUIDs — always discover them from a list call.

List endpoints are **paginated** (`page`, `limit`; `limit` max 100) and return `{items: [...], meta: {...}}`. A single call returns only one page — if you need every item, loop until you have covered `meta.totalItems`.

## Tool map

Group the tools mentally so you reach for the right one:

- **Deals & portfolios** — `list_deals`, `get_deal`, `create_deal`, `update_deal`, `move_deal`, `delete_deal`⚠, `list_deal_types`; `list_portfolios`, `create_portfolio`, `rename_portfolio`, `delete_portfolio`⚠; `bulk_import_deals`.
- **Documents & files** — `list_deal_documents`, `download_document`, `upload_file_to_folder`, `get_document_taxonomy`, `get_data_room_info`; `list_data_artifacts`, `get_data_artifact`; `initiate_file_veracity_check`, `get_file_veracity_status`.
- **Folders** — `list_deal_folders`, `create_folders`.
- **Workflows** — templates (`list_workflow_templates`, `get_workflow_template`, `get_deal_workflow_templates`, `set_deal_workflow_templates`); instances and tasks (`list_deal_workflows`, `get_workflow`, `list_deal_tasks`, `get_task`, `change_task_status`, `update_task_raci`); comments (`list_task_comments`, `create_task_comment`, `update_task_comment`, `delete_task_comment`⚠); `update_checklist_item`.
- **Access, roles, permissions** — `list_api_keys`, `get_documents_visible_to_user`, `set_user_resource_roles`, deny/FDR permission tools; org-role CRUD; permission checks.

⚠ marks destructive tools — see below.

## Workflow task statuses

A task's status is exactly one of:

`Open` · `In progress` · `Review` · `Done` · `On hold` · `Not applicable`

The API enforces a **transition graph** — you cannot jump to any status from any other. Before calling `change_task_status`, call `get_task` and read its `allowedNextStatuses` field; move only to a status listed there.

Observed transitions:

| From | Allowed next |
|---|---|
| Open | In progress, Done, On hold, Not applicable |
| In progress | Review, Done, On hold, Not applicable |
| Review | In progress, Done, On hold |
| Done | Open |

`Open → Review` is **not** a direct hop — go `Open → In progress → Review`. Whenever a target status is not directly allowed, walk the graph one step at a time.

Two field-name quirks that cause real bugs:

- `get_task` reports the current status in a field named `currentStatus` (not `status`).
- `change_task_status` takes the new value in a field named `new_status`.

**Reaching a status that is not directly allowed.** Walk the graph, bounded at ~4 hops. Each hop: re-read `currentStatus` from `get_task`; if it equals the target, stop; if the target is in `allowedNextStatuses`, go straight there; otherwise if `In progress` is allowed and you are not already in it, hop through it as a gateway; otherwise take the first allowed status. This reliably carries a task `Open → In progress → Review → Done` without hard-coding the graph.

## Workflow nuances

These come from actually driving the workflow API in earlier work — they are not obvious from the tool list:

- **Templates are attach-only through the API.** `set_deal_workflow_templates` attaches or detaches *existing* templates on a deal — there is no API call to create or edit a template itself. Templates are authored by Inveniam staff via Admin Settings (an XLSX upload). Don't promise programmatic template authoring.
- **RACI is role-bound in templates.** A template assigns each RACI slot to a *role* (e.g. Approver → Manager); the platform auto-resolves it to whoever holds that role on the deal. Changing a user's deal role can therefore change who is Responsible or Approver on its tasks.
- **`update_task_raci`'s `changes` payload is undocumented.** There is no published schema for it. Before calling it, read the `raci` field from `get_task` and mirror that structure rather than guessing.
- **No events or webhooks.** V2 has no push or subscribe primitive. To notice that a task changed status or received a new comment, you have to poll — `list_deal_tasks`, `get_task`, `list_task_comments`.

## Data-room write limitations

The API can **create folders** (`create_folders`) and **upload files** (`upload_file_to_folder`) — but it has **no move, rename, or delete** for folders or documents. If a user asks you to reorganize a data room (move documents between folders, rename a folder, delete files), explain that this is currently a UI-only operation; do not promise it through the tools.

Also:

- `update_deal` cannot change a deal's **price** — price is set only at deal creation.
- A deal's **description** caps near 1500 characters. Keep it under ~1300 to be safe.

## Document handling

Inveniam is designed as a virtualized access layer over a client's live data — its contract is that consumers do *not* keep copies of client files. When you `download_document`, treat the local file as ephemeral: hash it, parse it, extract what you need, then delete it. Don't build a persistent local cache of data-room documents. (A download may also arrive without a friendly filename — the tool falls back to the document UUID; pass `save_to` when you need a specific name.)

## Destructive and permission-changing tools

Some operations are destructive or change access: deleting a deal, portfolio, or comment (`delete_deal`, `delete_portfolio`, `delete_task_comment`), and the role and permission operations (`create_role`, `update_role`, `set_role_active`, `set_user_resource_roles`, and the deny/FDR permission ones), which make **org-wide or access-control changes**.

The rule is the same however you reach the API: **preview, get explicit user approval, then act.** Describe exactly what will be deleted or changed, wait for the user to approve that specific action, and only then make the call. *How* that rule is backed up differs by interface:

- **Direct REST API** — there is no safety gate; a `DELETE` request executes immediately. You must do the preview-and-confirm step yourself before issuing it.
- **The local `inveniam-mcp` wrapper** — implements a gate: its destructive tools take a `confirm=True` argument and return a harmless preview unless it is set. That flag is a feature of the wrapper, not of the platform — don't assume it exists on other interfaces.
- **An official or other MCP server** — check its own documentation for whether and how it gates destructive calls; don't assume.

Either way, the burden is on you to confirm intent before any destructive or permission-changing call.

A provisioning quirk worth knowing: issuing API keys is **not** a permission on the default **Manager** role — among the built-in roles it sits only on **Administrator**, which is far too broad to grant a counterparty or an external agent account. To give someone API access without making them an admin, assign them a custom role that carries the API-key permission, alongside whatever functional role they need (e.g. Manager on the deal). The built-in roles are org-wide and typically can't be edited, so an additive custom role is the way to keep the blast radius tight.

## Gotchas

- **Read the error body.** When a call fails, the error message includes the API's own explanation (for example, an invalid status value lists the valid ones). Use it to self-correct instead of guessing.
- **Workflow comments are author-scoped.** You can edit or delete only a comment that the *same authenticated user* posted. An administrator cannot modify another user's comment — the API returns HTTP 400. Plan comment cleanup around who originally posted each one.
- **File veracity is asynchronous.** `initiate_file_veracity_check` returns a `jobId`; poll `get_file_veracity_status` with it until the job resolves.
- **Comment payloads are loosely shaped.** Prior direct-API work found the comment list under any of `items`, `comments`, or `data`, and individual comments using `content` or `body` for the text and `id` or `commentId` for the identifier — inspect the actual response rather than hard-coding one key. Comments are plain Markdown with no structured/attestation field type, so any machine-readable data carried in a comment is formatted and parsed by convention.
- **Top-level vs reply comments.** A reply carries a `parentId`; a top-level comment has none — filter on that when you want only original comments.

## Common recipes

**Explore a deal**
`list_deals` → pick one → `get_deal` → `list_deal_folders` → `list_deal_documents`.

**Read what the platform extracted from a deal's documents**
`list_data_artifacts` (deal-scoped) → `get_data_artifact` for the document you care about. Pair with `get_document_taxonomy` to see how a document is classified.

**Review and comment on a deal's workflow**
`list_deal_workflows` → `list_deal_tasks` → `get_task` for detail → `create_task_comment` to post a note (pass `parentId` to reply to an existing comment).

**Advance a task**
`get_task` → read `allowedNextStatuses` → `change_task_status`, one step at a time toward the target status.

**Verify a file's integrity**
`initiate_file_veracity_check` (give it a `fileId`, `taxonomyId`, or `artifactId`) → poll `get_file_veracity_status` with the returned `jobId`.

**Check who can see what**
`get_documents_visible_to_user` (deal + user email), or `check_user_document_permissions` (user + specific document IDs), or `get_user_resource_permissions` (user + resource).
