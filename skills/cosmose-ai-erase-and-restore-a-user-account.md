---
name: cosmose-ai-erase-and-restore-a-user-account
description: >-
  Honour a KaiKai consumer's erasure request through the Cosmose AI forget-me operations, and restore an account that
  was deleted in error — including the undocumented reset cap that will eventually refuse you.
generated: '2026-08-11'
method: generated
source: openapi/cosmose-ai-deal-hunter-registration-api-openapi.yml
api: Cosmose AI Deal Hunter Registration API
base_url: https://api.sg.cosmose.co/deal-hunter-registration-api
operations:
  - deleteUserAccount
  - restoreUserAccount
  - unsubscribe
---

# Erase and restore a KaiKai user account

Cosmose AI exposes right-to-erasure as API operations rather than only as a support workflow — the contract carries a
`forget-me-controller` tag with a delete/restore pair. That is more than most behavioural-data companies publish. The
gaps below are equally real and are stated as gaps, not filled in.

## Before you start

- **Base URL:** `https://api.sg.cosmose.co/deal-hunter-registration-api` (not the internal host in the document's
  `servers[]`).
- **Auth:** global `bearerAuth` JWT.
- **This is destructive and not idempotent.** There is no `Idempotency-Key` on either operation. A retried `DELETE`
  after a timeout cannot be deduplicated by the service.
- **No retention window is published.** Nothing states how long a deleted account remains restorable, or when erasure
  becomes permanent. Do not promise a consumer a window you cannot cite.

## Steps

1. **Delete the account.** `DELETE /v1/user-account` (`deleteUserAccount`). Expect `404` carrying
   `errorCode: USER_NOT_FOUND` if the identity does not resolve — verify the installation id / phone number used at
   registration first. The de facto key of this graph is the installation, not a user id: `Installation-Id` appears
   as a header on token issuance and as the grouping key of `InvitationsByInstallationId`, but no schema ever declares
   it as a field, so its format is unstated.

2. **Stop marketing contact separately.** Deleting the account is not the same call as unsubscribing.
   `POST /v1/registration/unsubscribe` (`unsubscribe`), body `UnsubscribeDto`, is its own operation and its own
   record. If the consumer's request was "stop emailing me" rather than "erase me", use this one.

3. **Restore only if the deletion was an error.** `PUT /v1/user-account/restore` (`restoreUserAccount`), body
   `RestoreRequest`. This is capped: a `400` carrying `errorCode: MAX_RESETS_EXCEEDED` means the account has been
   restored more times than allowed. **The cap value is not published and there is no published recovery path once you
   hit it** — so treat every restore as potentially your last and confirm intent before calling.

## What to tell a consumer who asks

The public-facing opt-out surface is `https://cosmose.ai/opt-out`, and the privacy policy is at
`https://cosmose.ai/privacy-policy`. Both are routes in a client-rendered Angular app, so they are readable by a
person and not by a machine. No SOC 2, ISO 27001 or equivalent assurance is published — see
`conformance/cosmose-ai-conformance.yml`.
