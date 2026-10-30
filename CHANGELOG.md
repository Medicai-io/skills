# Changelog

## 1.0.0 (2026-11-09)

First public release of `medicai-integration`, submitted to Anthropic's directory.

- Every API flow verified against the live platform except webhooks, which follow the published specification
- Authenticated viewer embed verified end to end; origin whitelisting and the `ready-for-message` handshake documented
- Public demo viewer and local-files viewer documented as the no-account path
- Listing icon, discovery keywords and directory listing URLs

## 0.1.0 (2026-10-29)

First packaged version, private preview.

- Router SKILL.md with the two API layers, onboarding (15-day trial), no-account demo viewer, quickstart, eight rules, pitfalls table
- References: concepts, authentication (incl. workspace ids), viewer embed (demo viewer, embed + postMessage, gallery, public-study links), uploads (DICOM, files, cloud documents), cases and sharing (incl. trash/delete), webhooks, plans and limits, migrating from flat routes (existing integrations only)
- Synced to the October 2026 Medicai API: workspace-scoped routes everywhere, `X-Medicai-Client` mandatory, `searchInTrash` required, `withResources` on trash/delete, cloud documents from Markdown
- Four test cases in the layout that Claude Code's plugin evaluation command reads, one of which never mentions Medicai
