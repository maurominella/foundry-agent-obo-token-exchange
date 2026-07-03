# Changelog

All notable, reader-visible changes to this documentation are recorded here.
The format is loosely based on [Keep a Changelog](https://keepachangelog.com/),
and this project uses simple, dated version tags (`v1.0`, `v1.1`, …).

> **Stability note.** Filenames, chapter numbers, titles, and order are treated as a
> stable published interface (see *Maintaining this documentation* in the README).
> Changes are additive by default; any rename, reorder, split, or merge is called
> out explicitly below.

## [v1.3] — 2026-07-03

### Added
- **Appendix A — The Three Registered Applications** (`docs/11-appendix-app-registrations.md`): a per-application reference for the Bot, Foundry-access, and Downstream (App-OBO) registrations, organized by Entra blade — Overview, Certificates & secrets, Expose an API, and API Permissions — with the relevant code snippets and 11 screenshots (`docs/images/11-01-*` … `11-11-*`).
- Linked the appendix from the README table of contents and repository layout.

## [v1.2] — 2026-07-02

### Changed
- **Chapter 04 §(1D) — OBO via the Bot's OAuth Connection.** Reverted the OAuth Connection scope back to Foundry's `https://ai.azure.com/.default`, with a clearer explanation: on this Approach-1 path the connection mints a **user-token** toward Foundry (`aud = https://ai.azure.com`), which Foundry validates then strips, and which requires the **Foundry User** RBAC role on the project. The App-OBO scope belongs to the Token C discussion ([Chapter 07](docs/07-token-c-user-assertion.md)), not to §(1D). Replaced the §(1D) OAuth Connection screenshot accordingly (`docs/images/04-05-azure-bot-oauth-connection-foundry.png`).
- **Chapter 07 §7.2 (Point 1) — Token C scope.** Added the explanation that the App-OBO scope (`api://app-obo/…/access_as_user`) makes Entra place `aud = api://app-obo/…` and `scp = access_as_user` in Token C, enabling the agent's downstream OBO; reworded the "what if we used Foundry as the scope?" note. Replaced the Token C OAuth Connection screenshot (`docs/images/07-01-oauth-connection-token-c-settings.png`).
- **Chapter 04 §4.1** — Minor wording in the Approach-1 introduction.

## [v1.1] — 2026-07-02

### Changed
- **Chapter 04 §(1D) — OBO via the Bot's OAuth Connection.** Corrected the scope the OAuth Connection requests: it targets the **App-OBO** application (`api://app-obo/3a0fad96-b026-4f5f-914a-fc6348656f6b/access_as_user`), **not** Foundry (`https://ai.azure.com/.default`). Consequently the outgoing token now carries `aud = api://app-obo/3a0fad96-…` and `scp = access_as_user`, so the Foundry Hosted Agent can perform the downstream OBO exchange (e.g. toward Microsoft Graph).
- Reworded the *Token Exchange URL* and `client_id` bullets in §(1D) accordingly, and **replaced the OAuth Connection screenshot** (`docs/images/04-05-azure-bot-oauth-connection-foundry.png`) with the corrected one.

## [v1.0] — 2026-07-01

### Added
- Initial publication of the full documentation set, built from the validated
  end-to-end source (dev environment, secrets masked).
- Introduction and navigation (`README.md`), including the maintenance policy.
- **Final Result** page (`docs/00-final-result.md`) — the silent, end-to-end
  Teams → bot → Foundry agent → Microsoft Graph round-trip, on behalf of the user.
- Chapters 01–10 covering the Bot Service and the Activity Protocol limit, the
  authentication flow and prerequisites, the three Entra app registrations,
  Token A (Teams SSO), Token B (Foundry access — user-delegated vs app-only),
  the two identity axes, the five-token model, Token C (the user assertion),
  Token D (the agent-side downstream OBO), publishing to Teams, and the code
  wiring plus token verification.
- 16 screenshots under `docs/images/`, referenced from their chapters.
