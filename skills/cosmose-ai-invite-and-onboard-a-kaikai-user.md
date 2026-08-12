---
name: cosmose-ai-invite-and-onboard-a-kaikai-user
description: >-
  Issue a KaiKai invitation, register a consumer against it, confirm the registration, and exchange the result for a
  bearer token — the full invite-driven onboarding path on the Cosmose AI Deal Hunter Registration API.
generated: '2026-08-11'
method: generated
source: openapi/cosmose-ai-deal-hunter-registration-api-openapi.yml
api: Cosmose AI Deal Hunter Registration API
base_url: https://api.sg.cosmose.co/deal-hunter-registration-api
operations:
  - createInvitation
  - getInvitationUsage
  - register
  - confirmRegistration_1
  - generateTokens
---

# Invite and onboard a KaiKai user

Every operationId below was read out of the provider's published OpenAPI. Nothing here is invented; where the
provider documents nothing, this skill says so rather than filling the gap.

## Before you start

- **Base URL:** `https://api.sg.cosmose.co/deal-hunter-registration-api`. Do **not** use the `servers[]` value in the
  provider's own document — it advertises `http://deal-hunter-registration-api/deal-hunter-registration-api`, an
  internal cluster name that does not resolve for any external caller.
- **Auth:** every operation inherits a global `bearerAuth` (HTTP bearer, JWT). There is no public key-issuance flow;
  access is partner-provisioned. See `authentication/cosmose-ai-authentication.yml`.
- **No idempotency.** There is no `Idempotency-Key` on any operation. Assume a retried write can duplicate. Treat
  `USER_ALREADY_REGISTERED` and `EMAIL_ALREADY_TAKEN` as your only collision signals.
- **No 401 in the contract.** The document declares 200/400/403/404 only, but the gateway returns HTTP 401 with a
  *different* envelope (`{"error": ..., "error_description": ...}`) than the documented `ErrorDto`
  (`{errorCode, message}`). Handle both shapes.

## Steps

1. **Issue an invitation.** `POST /v1/invitations` (`createInvitation`), body `InvitationRequestDto`, returns
   `InvitationDto` with a `localizedMessage`. This is quota-limited: a `400` carrying
   `errorCode: INVITATION_DAILY_LIMIT_EXCEEDED` means you have exhausted the per-day allowance. The allowance value
   is not published anywhere — you cannot pace against it, so back off for the day on that code.

2. **Check invitation usage before re-issuing.** `GET /v1/invitations` (`getInvitationUsage`) returns
   `InvitationUsageDto`; `PUT /v1/invitations` (`getInvitationTimestamps`) returns
   `InvitationsByInstallationId`, invitations grouped by installation. Both are reads — the second one is a read
   modelled as a `PUT`, which is a quirk of this contract, not a typo here.

3. **Register the consumer.** `POST /v2/register` (`register`), body `RegisterDto` — a single `phone_number` field
   constrained to `^\+?[\d \-]{8,17}$`. Expect `400 CODE_NOT_FOUND` if the invitation code is unknown,
   `400 CODE_EXCEEDED` if it has been fully redeemed, and `400 USER_ALREADY_REGISTERED` on a repeat.

   For the KaiKaiNow variant use `POST /v2/kkn-register` (`register_1`) instead — same shape, separate controller.

4. **Confirm the registration.** `POST /v2/confirm-registration` (`confirmRegistration_1`), body
   `RegisterConfirmRequest`, returns `RegisterConfirmResponse`. The KaiKaiNow twin is
   `POST /v2/kkn-confirm-registration` (`confirmRegistration`). Note the operationIds are crossed relative to what you
   would guess — the `_1` suffix is on the *standard* register and on nothing else — so bind by path, not by name.

5. **Exchange for tokens.** `POST /v2/token` (`generateTokens`). Parameters are query, not form-encoded:
   `grant_type` (required), `refresh_token`, `scope` (defaults to `kaikai`), plus an optional `Installation-Id`
   header that binds the token to the app installation. Returns `TokenInfo`: `access_token`, `refresh_token`, `type`,
   `expires_in`. This looks like OAuth 2.0 and is not — there is no oauth2 security scheme, no authorization
   endpoint, and no discovery document.

## If the user cannot be invited

Fall back to `POST /v1/registration/wait-list` (`registerNewUserWithoutCodeOnTheWaitingList`) with
`RegisterNewUserNoCodeDto`. An operator with a privileged token then approves them via
`PUT /v1/registration-admin/approval` (`approveUsers`, body `ApproveUsersDto`). A `403 FORBIDDEN` on that call means
your token is not an admin token; the contract has no scope or role model to tell you which one is.
