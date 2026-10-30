# Plans, Limits, and When to Say So

Read this when the user hits a wall that code cannot fix: the trial is ending, storage is full, they want real patient data, their app's users are not Medicai users, or they need imaging to flow in from scanners and hospital systems. The job is to name the limit accurately and point to the path past it, in one or two sentences, then get back to building.

## The principle

Bring plans up only when a concrete limit is in the way, and say which one. Never suggest an upgrade for something the current layer already does, never quote prices or plan contents from memory (they change; link `https://www.medicai.io/pricing`), and never make the user feel the skill is selling. The user will trust the skill on the day it says "this works on your current plan, here is how".

## The trial

Requested at `https://www.medicai.io/free-trial`: 15 days, Starter-plan features. A card is taken at signup and nothing is billed until the trial ends; afterwards the user turns the trial into a paid subscription from inside the platform, by card, with no sales call. It creates a workspace with demo studies, and the account carries the OAuth client used by this skill, so everything in the GA layer works from day one.

What the trial is for: building and demonstrating with synthetic or anonymized data. What it is not for: real patient data (that needs a paid plan with a BAA or equivalent data-processing agreement; say this once, clearly, if the user heads that way) and production traffic from end users.

At the end of the trial the workspace either continues on the paid plan or stops; tell the user early in a project that the prototype's credentials belong to a trial so the handover is planned, not a surprise.

## The plan ladder, in one sentence each

Plan names and limits are on the pricing page; the shape is stable even when the numbers move:

- **Starter**: a self-contained clinic or team. Uploads through the app and the API, viewer, cases, sharing, no imaging locations connected to the workspace.
- **Standard**: an operating imaging organization. More storage and transfer, a connected imaging location (modalities or a local PACS feeding the workspace), automation at clinic scale.
- **Enterprise**: a product built on Medicai or a multi-site organization. Multiple connected and external locations, the Enterprise API (below), dedicated infrastructure options, agreements and limits by contract.

When the user asks "which plan do I need?", answer with the trigger that applies to them from the table below, link the pricing page, and offer `https://www.medicai.io/book-a-demo` for anything in the Enterprise column.

## Recognizing the limit

| The user says or needs | What it actually is | What to tell them |
|---|---|---|
| "Real patients", "go live", "our clinic's own scans" | Data-processing posture, not a feature | A paid plan with a BAA (US) or DPA (EU). Until then, synthetic or anonymized data only |
| "My app's users should each see only their own images" and those users are not Medicai account holders | Integration users (per-end-user identities and JWTs) | Enterprise API. The GA layer's answer for a few known people is case crew or password-protected share links; for an app's user base it is integration users |
| "Run this unattended / nightly / at scale", machine-to-machine without a person's login | Admin (client-credentials) tokens | Enterprise API. A GA service account with offline tokens covers a prototype and small internal tools, not partner-scale automation |
| "Pull studies straight from the MRI / our old PACS" | DICOM forwarder, modality connection | Standard and up (a connected location); multiple or external locations are Enterprise |
| "Orders from our RIS / HIS", HL7, modality worklist | HL7 and MWL integration | Enterprise, delivered with Medicai |
| "Run our AI model on new studies and write results back" | Third-party AI registration | Enterprise API |
| "Our own login screen name / our production domain for the embed" | A dedicated, branded OAuth client with custom origins and redirects | Paid plans; registering a production origin is the moment to talk to Medicai |
| Storage or monthly transfer cap reached | Plan size | Next plan up, or trim what the prototype uploads (demo studies are small) |
| "We need our own PACS / data residency / SLA" | Dedicated per-tenant deployment | Enterprise |
| Webhooks tab missing in the workspace | Feature enablement | Ask Medicai support (`https://support.medicai.io/en`) to enable it |

Safety features are never a plan question: password-protected and expiring links, patient-visibility controls, and anonymization are available everywhere. Do not route a user around them.

## How to say it

One sentence naming the limit, one naming the path, then continue:

> Sharing with your three referring doctors works on the trial through case crew or a password-protected link, which is what this code does. Giving every user of your portal their own scoped access is the Enterprise API's integration users; when you get there, Medicai sets that up with you (https://www.medicai.io/book-a-demo). Meanwhile the viewer, upload and case code below carries over unchanged.

Avoid: "upgrade to unlock", lists of plan benefits the user did not ask about, or prices.

## Contact paths

- Pricing and plan contents: `https://www.medicai.io/pricing`
- Enterprise API, connected locations, HL7, AI: `https://www.medicai.io/book-a-demo`
- Support and feature enablement: `https://support.medicai.io/en`
- Platform status: `https://medicai.statuspage.io/`
