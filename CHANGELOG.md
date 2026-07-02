# Changelog

All notable, reader-visible changes to this documentation are recorded here.
The format is loosely based on [Keep a Changelog](https://keepachangelog.com/),
and this project uses simple, dated version tags (`v1.0`, `v1.1`, …).

> **Stability note.** Filenames, chapter numbers, titles, and order are treated as a
> stable published interface (see *Maintaining this documentation* in the README).
> Changes are additive by default; any rename, reorder, split, or merge is called
> out explicitly below.

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
