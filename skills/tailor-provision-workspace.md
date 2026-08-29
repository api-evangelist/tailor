---
name: tailor-provision-workspace
description: Provision a Tailor Platform workspace and deploy an application into it, using the published OperatorService control-plane RPCs.
api: Tailor Platform Operator API
operations:
  - ListAvailableWorkspaceRegions
  - CreateWorkspace
  - GetWorkspace
  - ListWorkspaces
  - CreateApplication
  - GetApplication
  - GetApplicationSchemaHealth
generated: '2026-08-29'
method: generated
source: grpc/tailor-tailor-v1-service.proto
---

# Provision a Tailor workspace and application

Every operation below is a real RPC on `tailor.v1.OperatorService`, published at
`github.com/tailor-inc/proto` and callable over gRPC or plain HTTP via ConnectRPC.
Base host: `https://api.tailor.tech`.

## Before you start

- Authenticate. Either a platform OAuth 2.0 access token (authorization_code + PKCE
  `S256`, or client_credentials) or a personal access token prefixed `tpp_`, sent as
  `Authorization: Bearer <token>`.
- Every RPC in this flow can return `Unauthenticated` (token missing, expired, invalid)
  and `InvalidArgument` (request invalid). Handle both before anything else.

## Steps

1. **Pick a region.** Call `ListAvailableWorkspaceRegions`. This RPC declares
   `idempotency_level = NO_SIDE_EFFECTS`, so it is safe to retry.

2. **Create the workspace.** Call `CreateWorkspace` with a name matching
   `^[a-z0-9][a-z0-9-]{1,61}[a-z0-9]$` and the region from step 1.

   `CreateWorkspace` does **not** declare `IDEMPOTENT` and Tailor supports no
   idempotency key. Do not blind-retry it. If the call times out, call `ListWorkspaces`
   and check whether the workspace exists before trying again.

3. **Confirm.** Call `GetWorkspace` with the returned id. Safe to retry.

4. **Turn on delete protection** if this workspace matters. `Workspace` carries a
   `delete_protection` boolean. Setting it is the only pre-emptive guard Tailor offers;
   `RestoreWorkspace` exists but no restore window is documented anywhere, so do not
   plan on it.

5. **Create the application.** Call `CreateApplication`. `domain`, `url`, `create_time`,
   `update_time`, `create_user_id` and `update_user_id` are `OUTPUT_ONLY` — do not send
   them. Set `cors` and `allowed_ip_addresses` if the app is browser-facing, and
   consider `disable_introspection: true` for production.

6. **Verify schema health.** Call `GetApplicationSchemaHealth` before pointing anything
   at the generated GraphQL API. Safe to retry.

## Error handling

| Status | Meaning here | Do |
|---|---|---|
| `Unauthenticated` | Token missing, expired, invalid | Re-authenticate, then retry |
| `InvalidArgument` | Failed a protovalidate constraint | Fix the payload; the constraints are inline in the .proto |
| `AlreadyExists` | Name is taken in this scope | Choose another name, or switch to the `Update*` RPC |
| `PermissionDenied` | You can see it but may not do this | Needs an access grant, not a retry |
| `NotFound` | Missing **or** not visible to you | Tailor collapses both — check the id AND your access |
| `FailedPrecondition` | System not in the required state | Resolve what the message names first |

## Notes

- No dry-run parameter exists on any RPC. To rehearse, drive this through the
  `tailor-platform/tailor` Terraform provider and run `terraform plan`.
- `tailor api list` enumerates every invocable OperatorService method and
  `tailor api inspect` prints an endpoint's input message tree — use them instead of
  guessing a field name.
