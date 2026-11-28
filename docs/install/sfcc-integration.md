# Installing the SFCC Integration (OCAPI)

Bridge's Salesforce B2C Commerce (SFCC) tools give an AI coding agent read access to
a sandbox's object model, custom object definitions, and site preferences — plus a
small set of sandbox-only writes — through the **OCAPI Data API**. This guide covers
setting up the OCAPI client that those tools authenticate against.

> **Sandbox / local development only.** This integration is intended for a **developer
> sandbox**, and that restriction is **enforced in code**: before any SFCC tool runs,
> Bridge validates the hostname your credentials actually resolve to — from `dw.json`
> or `SFCC_*` — against the sandbox forms listed below. An unrecognized host is refused
> with a `403` (`error.code: "TARGET_NOT_SANDBOX"`) before any request leaves your
> machine. The check reads the resolved hostname, never the `instance` tool argument,
> so omitting `instance` or passing `"sandbox"` cannot bypass it.
>
> Accepted sandbox hostname forms:
>
> - `<realm>-<nnn>.sandbox.<region>.dx.commercecloud.salesforce.com`
> - `<realm>-<nnn>.sandbox.dx.commercecloud.salesforce.com`
> - `<realm>-<nnn>.dx.commercecloud.salesforce.com`
>
> Anything else — a `production-`/`staging-`/`development-` prefixed host, or any
> `*.demandware.net` host — is rejected.
>
> Still do not configure the grants below on an instance that holds real data.
> Credentials stay local (in `dw.json` or `SFCC_*` env vars) and are never sent to
> Bridge.

For the full per-tool list and what each SFCC tool depends on, see
[MCP Tool Integration Dependencies](./mcp-tool-integrations.md). For the tool reference
and the `BRIDGE_MCP_PROFILE` gating, see the SFCC section of the
[package README](../../README.md).

## Prerequisites

- A running SFCC **developer sandbox** and its hostname
  (e.g. `zzzz-001.sandbox.us01.dx.commercecloud.salesforce.com`).
- An **Account Manager API client** — a `client-id` and `client-secret`. This is the
  OCAPI client the tools use to obtain an OAuth token. Create one in Account Manager
  (**API Client** → *Add API Client*) if you don't already have it, and note its
  `client_id`.
- Business Manager access to the sandbox with permission to edit **Open Commerce API
  Settings**.

## 1. Grant the OCAPI client access in Business Manager

In Business Manager for the sandbox:

**Administration → Site Development → Open Commerce API Settings → Data API** tab.

Add the client entry below to the `clients` array of the Data API settings, then
**Save**. It grants only the resource families and HTTP methods Bridge's SFCC tools
actually call — not a global `/**` grant. `check_permissions` prints the same JSON on
a 401/403, split into the two blocks.

**READ/SEARCH TOOL GRANTS** — required by the `sfcc` read tools. (`post` is OCAPI's
convention for its `*_search` endpoints, not a mutation.)

```json
{
  "client_id": "<your-client-id-here>",
  "resources": [
    { "resource_id": "/system_object_definitions",    "methods": ["get"],         "read_attributes": "(**)", "write_attributes": "(**)" },
    { "resource_id": "/system_object_definitions/**", "methods": ["get", "post"], "read_attributes": "(**)", "write_attributes": "(**)" },
    { "resource_id": "/site_preferences/**",          "methods": ["get", "post"], "read_attributes": "(**)", "write_attributes": "(**)" },
    { "resource_id": "/custom_object_definitions/**", "methods": ["get", "post"], "read_attributes": "(**)", "write_attributes": "(**)" }
  ]
}
```

**MUTATION GRANTS** — required **only if you enable `BRIDGE_MCP_PROFILE=sfcc-write`**,
which registers the nine destructive write tools. These are shipped capabilities, not
future work. No `delete` is granted, because no shipped write tool performs one; the
`get` entries are needed for the If-Match ETag round trip that precedes each `PATCH`.

```json
{
  "client_id": "<your-client-id-here>",
  "resources": [
    { "resource_id": "/system_object_definitions",    "methods": ["get"],                  "read_attributes": "(**)", "write_attributes": "(**)" },
    { "resource_id": "/system_object_definitions/**", "methods": ["get", "put", "patch"],  "read_attributes": "(**)", "write_attributes": "(**)" },
    { "resource_id": "/custom_object_definitions/**", "methods": ["get", "put", "patch"],  "read_attributes": "(**)", "write_attributes": "(**)" },
    { "resource_id": "/sites/*/site_preferences/**", "methods": ["get", "patch"],         "read_attributes": "(**)", "write_attributes": "(**)" }
  ]
}
```

Notes:

- Value writes (`site_preference_values_set`) are site-scoped: they need the mutation
  resource `/sites/*/site_preferences/**`. The read/search grant keeps the org-level
  `/site_preferences/**`. **If you pasted an earlier mutation grant, it lacks the
  site-scoped resource.** Replace `<YOUR_CLIENT_ID>` with your client id in the newly
  printed grant (`check_permissions` prints it) and paste it again before running
  value writes.
- `check_permissions` probes a GET only. It shows read access, not write access.
- The `client_id` **must match** the Account Manager API client whose credentials you
  put in `dw.json` / `SFCC_*` below. Replace the value above with your own client id if
  it differs.
- If the Data API settings are empty, wrap the entries in the standard settings
  envelope. Merge the resource lists from the block(s) above into one `resources`
  array — do not substitute a global `"resource_id": "/**"` grant:

  ```json
  {
    "_v": "23.2",
    "clients": [
      {
        "client_id": "<your-client-id-here>",
        "resources": [
          { "resource_id": "/system_object_definitions",    "methods": ["get"],         "read_attributes": "(**)", "write_attributes": "(**)" },
          { "resource_id": "/system_object_definitions/**", "methods": ["get", "post"], "read_attributes": "(**)", "write_attributes": "(**)" },
          { "resource_id": "/site_preferences/**",          "methods": ["get", "post"], "read_attributes": "(**)", "write_attributes": "(**)" },
          { "resource_id": "/custom_object_definitions/**", "methods": ["get", "post"], "read_attributes": "(**)", "write_attributes": "(**)" }
        ]
      }
    ]
  }
  ```

- `check_permissions` (below) prints a ready-to-paste grant JSON on a 401/403, so you can
  also let the tool tell you exactly what to add.

## 2. Provide credentials locally

Create a `dw.json` in your project root (auto-added to git exclude — never commit it):

```json
{
  "hostname": "zzzz-001.sandbox.us01.dx.commercecloud.salesforce.com",
  "client-id": "<your-client-id-here>",
  "client-secret": "<account-manager-client-secret>"
}
```

Accepted key spellings: `hostname`/`host`, `client-id`/`clientId`/`client_id`,
`client-secret`/`clientSecret`/`client_secret`. A single config is simplest. With a
multi-entry `configs[]` array, pass one of the listed sandbox hostnames as `instance`
and that config's credentials are used (an unapproved host is refused before any
credential lookup); without an `instance` the call fails as `multi-sandbox`. Alternatively, export
`SFCC_HOSTNAME` / `SFCC_CLIENT_ID` / `SFCC_CLIENT_SECRET` in the MCP server environment.

## 3. Set the repo `version` config field

Set the repo's `version` config to your SFCC project type — one of
`sfra | pwakit | sitegenesis | storefrontnext | hybrid`. The call-time gate reads this;
a non-SFCC value blocks the OCAPI tools (the eight reads, the nine writes, and
`check_permissions`). It does not block `sfcc_setup_status` or `sfcc_log_query` — log
query has its own backend gate. Set it via your
normal config path, the `config_field` MCP tool (operation `update`, field `version`),
or the `/teach-bridge` skill.

## 4. Enable the SFCC tools

No SFCC tool is registered by default — a session on the default `core` profile has
none. Every SFCC tool sits behind **two independent profile groups**:

| Group | Registers |
|---|---|
| `sfcc` | the two diagnostics (`sfcc_setup_status`, `check_permissions`), the 8 OCAPI read tools, and `sfcc_log_query` — read-only |
| `sfcc-write` | the 9 destructive write tools |

Neither implies the other. An `sfcc-write`-only session has no diagnostics, so it
cannot run `check_permissions` to print the grant JSON; use `sfcc,sfcc-write` when
you need writes. Add what you need to `BRIDGE_MCP_PROFILE` in the MCP server
`env` block (it is comma-separated), then **restart the MCP client**:

```json
"env": { "BRIDGE_MCP_PROFILE": "sfcc" }
```

For reads plus writes, use `"sfcc,sfcc-write"`. `full` expands to every group and is
therefore write-capable.

> **Migration.** `sfcc` used to register the nine write tools too. It no longer does.
> If you were relying on SFCC writes through `BRIDGE_MCP_PROFILE=sfcc`, change it to
> `BRIDGE_MCP_PROFILE=sfcc,sfcc-write`. `full` users keep write access and need no
> change.

## 5. Verify

Ask your agent to run:

1. `sfcc_setup_status` — expect all prerequisite checks ✓ (Bridge API key, repo name,
   `version` config, `dw.json` presence/uniqueness, AM/OCAPI token acquisition).
2. `check_permissions` — probes OCAPI via `GET /system_object_definitions`. A 200 (with
   the OCAPI version) confirms the grant. On 401/403 it prints the exact grant JSON to
   paste back in step 1.

Restart the MCP client after any credential, grant, or env change — a running session
does not pick them up.

## Notes

- **WebDAV logs are separate.** `sfcc_log_query` authenticates with a Business Manager
  username + a 40-character **WebDAV access key** over HTTP Basic auth — *not* the OCAPI
  OAuth token configured here. `sfcc_setup_status` reports OCAPI (step 5) and WebDAV
  (step 6) independently; one can be green while the other is not.
- **Writes are sandbox-only.** The write tools (attribute/preference create/update) target
  a developer sandbox and echo a paste-ready grant JSON on a 403.
