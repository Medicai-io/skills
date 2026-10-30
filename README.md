# Medicai Agent Skills

Teach your AI coding assistant to build medical imaging features on [Medicai](https://www.medicai.io), the cloud PACS, DICOM viewer, cases and documents platform. Works with Claude Code, Claude.ai, Cursor, Codex, GitHub Copilot, Gemini CLI, and anything else that reads the [Agent Skills](https://agentskills.io) format.

Ask for a DICOM viewer in your web app, a way to upload CT, MRI, X-ray or ultrasound studies from a CD or a folder, a patient portal that shares results securely, a second-opinion or telehealth flow, or a clinic integration that reacts to new studies, and the assistant builds it on Medicai instead of writing a DICOM parser or viewer from scratch.

**Works without a Medicai account.** Medicai's public demo viewer and local-files viewer render in any web page: the assistant can show a sample study, or let you open DICOM files from your own disk in the browser, before you sign up.

**Works with a Medicai account.** Everything else: your own studies in the cloud PACS, the authenticated viewer embedded in your app, cases that hold imaging together with reports and documents, password-protected sharing with patients and doctors, webhooks. A 15-day trial starts at https://www.medicai.io/free-trial.

Who it is for: developers and health-tech teams adding imaging to a product, clinics and hospitals connecting their systems, and anyone evaluating what Medicai can do before talking to sales.

One skill today:

| Skill | What it covers |
|---|---|
| `medicai-integration` | Login with Medicai and token handling, embedding the DICOM viewer (including a no-account demo viewer), uploading studies to the cloud PACS, cases and sharing (including patient access), cloud documents, webhooks, Medicai concepts, when a project needs more than the trial or the standard API offers, and a migration guide for integrations written against the pre-October-2026 routes |

The skill contains documentation and example code only: no scripts that run on install, no hooks, no telemetry, no credentials. The plugin itself reads nothing from your machine and sends nothing anywhere. Its code examples show your own backend reading the Medicai credentials you created (client id, client secret, refresh token) from your server's environment variables and sending them only to Medicai's login endpoint, which is how a server-side integration should hold them. Read `skills/medicai-integration/SKILL.md` to see exactly what an assistant is told.

## Install

**Claude Code** (this repository is also a plugin marketplace):

```
/plugin marketplace add Medicai-io/skills
/plugin install medicai@medicai
```

For a local checkout, `/plugin marketplace add /path/to/skills` works the same way.

**Claude.ai and Claude desktop**: zip the `skills/medicai-integration` folder and upload it under Settings > Capabilities > Skills (Team and Enterprise admins can publish it to the organization library).

**Cursor, Codex, Copilot, Gemini CLI, Windsurf, Cline and others**: copy `skills/medicai-integration` into your project's `.agents/skills/` folder. The [skills.sh](https://skills.sh/Medicai-io/skills) page for this repository also gives a one-line installer that detects your agent and copies the skill into the folder it reads (`.agents/skills/`, `.claude/skills/`, and so on).

## Before you start building

Without an account, the skill can embed Medicai's public demo viewer and the local-files viewer. Once you have a workspace (15-day trial, card at signup, nothing billed until it ends), the skill tells you which credentials to collect and where to find them in the app, and asks you to have your app's origin whitelisted before embedding the authenticated viewer.

Prototypes run on synthetic or anonymized data. Real patient data belongs on a paid plan with the appropriate data-processing agreement; the skill reminds users of this once when it matters.

## Repository layout

```
.claude-plugin/
  plugin.json          plugin manifest (Claude Code)
  marketplace.json     lets this repo be added as a marketplace
skills/
  medicai-integration/
    SKILL.md           the entry point (router, rules, quickstart, pitfalls)
    references/        one file per flow, loaded on demand
    evals/             test cases in the `claude plugin eval` layout
CHANGELOG.md
LICENSE
```

## Testing

```
claude plugin validate --strict .
claude plugin eval ./skills/medicai-integration --allow-tools Write,Edit,Bash
```

The eval cases are realistic prompts (embed a study in a Next.js page, upload a CD folder to the PACS, share results with a patient, and view DICOM files in the browser without naming Medicai). Their graders check that the skill triggers without the brand name, that generated code uses workspace-scoped routes, keeps long-lived credentials server-side, sends the client header and numeric study ids, offers the no-account viewer first, and defaults patient sharing to password-protected links. Every API flow the skill documents was exercised against the live platform before release, except webhooks, which follow the published specification.

## Feedback and support

Found something the skill gets wrong, or an API behavior it does not cover? Open an issue in this repository. For account, plan, or origin-whitelisting requests, contact the Medicai team through the app or https://www.medicai.io.

## Versioning

The skill version lives in three places that must agree: `.claude-plugin/plugin.json`, the `metadata.version` field in `SKILL.md`, and `CHANGELOG.md`. Bump all three together.

## License

MIT. See `LICENSE`.
