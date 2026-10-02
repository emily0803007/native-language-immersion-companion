# Native-language Immersion Companion

An open-source, self-hosted learning journal for native-language conversations with a beginner-level target-language companion in ChatGPT.

**Status: complete design documentation, not a released or tested application.** This repository documents the implementation plan. It does not yet contain application code, a Docker package, or a working demo. Platform facts were checked on October 2, 2026; account capabilities must still be tested.

> Talk in your native language. Get simple replies in your target language. Save selected practice, review what you learned, and reuse it tomorrow.

## Demo

<!-- TODO: replace this block with real screenshots after implementation. -->
[Demo screenshot placeholder: ChatGPT practice + daily learning report]
[Demo recording placeholder: save a phrase, review a report, start tomorrow's practice]

Do not present mockups as a working demo.

## Core concepts

- **Low-pressure immersion:** v1 targets Chinese-speaking beginners learning English from pre-A1 to A1. The learner can start in Chinese; the companion replies in simple English about the current scene.
- **Selected learning evidence:** the journal stores intentionally submitted practice and learning items, not an automatically reconstructed ChatGPT chat log.
- **Learner-owned storage:** selected records, vocabulary, reports, and progress live in the learner's deployment. ChatGPT still processes the conversations and any records sent back to it. Self-hosted does not mean offline or OpenAI-free.
- **Interactive analysis:** the learner asks ChatGPT to analyze saved evidence and return a report. If write tools work on the account, ChatGPT submits it through MCP. Otherwise the learner copies the report JSON into the local dashboard.
- **Reuse before expansion:** later practice brings back familiar phrases in new scenes. A2 is a proposed learning path, not an automatic certification.

## Proposed architecture

```text
Learner <----> ChatGPT web
               | conversation / images / voice, where available
               | explicit MCP reads and approved writes
               v
       HTTPS MCP endpoint or eligible Secure MCP Tunnel
               |
       Self-hosted application
       +-- MCP tools + authentication
       +-- report validation + learning rules
       +-- local database + import staging
       +-- private dashboard
               ^
               | manual selected export import / report JSON paste
Learner -------+
```

The backend does not spend ChatGPT subscription credits through an API. Model analysis happens in an interactive ChatGPT session. No background LLM analysis is promised. Subscription usage limits still apply; infrastructure may cost money. Tunnel costs and eligibility have not been established.

## Setup and validation plan

### 1. Check the account before building

Use ChatGPT web and verify developer-mode access. Official developer documentation lists Plus and Pro among eligible plans and describes read/write tools; the Help Center currently gives narrower write eligibility. Test a read tool and a harmless, explicit write on your actual account. Do not advertise Plus/Pro write compatibility until verified.

### 2. Prepare the self-hosted deployment

There is no runnable Docker package in this documentation-only repository yet. Do not run `docker compose up` here: no Compose file or image has been supplied. The implementation must provide these pieces before installation can be claimed to work:

1. A server exposing Streamable HTTP MCP and the dashboard's private HTTP API.
2. A database migration step and persistent data volume, separate from the container image.
3. Owner authentication, a loopback-bound dashboard, and authenticated MCP access.
4. Environment configuration for dashboard origin, external MCP origin, authentication, and data directory. Never commit real secrets.
5. Health/readiness checks, database backup, restore instructions, and an account-specific integration test.

For a local pilot, keep the dashboard on the user's machine and connect MCP through an eligible tunnel or an authenticated HTTPS endpoint. Test persistence across a container restart and a restored backup before importing personal history. Exact build/start commands must be written against the real implementation, not invented in a design document.

### 3. Make MCP reachable from ChatGPT

A Docker service on `localhost` is not directly reachable by ChatGPT. Choose:

- A stable, authenticated HTTPS MCP endpoint reachable from ChatGPT; or
- OpenAI Secure MCP Tunnel, only if your Platform organization and ChatGPT account/workspace have the required access. The tunnel requires separate setup and runtime credentials.

Do not assume a ChatGPT subscription automatically includes Platform tunnel permissions. The exact setup commands remain TODO until an integration test is recorded.

### 4. Connect in developer mode

1. Enable developer mode in ChatGPT web. The developer guide currently points to Settings > Security and login; the Help Center shows a different settings route. Follow the route present in your account.
2. Open ChatGPT Plugins and create a developer-mode app for the MCP endpoint, or select an eligible tunnel. No public-directory submission is part of this private pilot.
3. Authenticate, review available tools, and select the app in a conversation.
4. Run `get_daily_plan`, then explicitly save a small practice sample. Check the write confirmation and dashboard receipt.
5. Ask for a report based on saved evidence. Save through `save_report`, or paste its JSON into the local dashboard if MCP writes are unavailable.

### 5. Start practice

Suggested conversation instruction, not an enforcement mechanism:

```text
I speak Chinese and am beginning English. I may write in Chinese.
Reply in simple A1-oriented English about my current situation.
Use familiar words and short sentences. Explain briefly in Chinese if I ask.
Invite a small English response when I am ready; do not interrupt every turn.
Read my daily plan when I ask. Save only learning items or practice samples
that I intentionally submit. Do not collect unrelated conversation history.
At the end, analyze saved evidence and show me a draft learning report.
```

Images are optional visual aids generated inside ChatGPT when available. Voice/MCP integration is unverified because official pages currently conflict. v1's required loop is text-based. Do not promise audio, transcripts, or generated images will automatically appear in the backend.

## Tool contracts and stored data

| Tool | Access | Purpose |
| --- | --- | --- |
| `start_session` | Write | Create a learning session and return its server-issued ID. |
| `log_turn` | Write | Save one intentionally submitted practice sample, not every chat turn. |
| `save_learning_item` | Write | Save a word, phrase, or pattern with evidence references. |
| `get_learning_evidence` | Read | Read selected samples with pagination and a source hash. |
| `get_report` | Read | Read a report for a local learning date and version. |
| `save_report` | Write | Validate and save a report tied to its evidence snapshot. |
| `get_daily_plan` | Read | Return due review, familiar phrases, a scene, and selection reasons. |

The server derives identity from authentication, validates ownership, and rejects unrecognized fields. Write requests carry a client operation ID. An identical retry returns the saved result; a changed payload under the same ID returns a conflict. A write receipt contains a record ID, content hash, saved time, and version. A schema-valid report still needs evidence validation.

Stored entities are users, learning sessions, selected practice samples, learning items, item evidence, review events, mastery state, report versions, daily plans, import batches, and idempotency operations. Timestamps are stored in UTC; learning dates use the learner's configured timezone. Original text is preserved alongside normalized matching keys. Display, imitation, prompted production, and independent recall are distinct evidence types.

## Daily learning loop

1. Ask for the daily plan and practice around the current scene.
2. Intentionally save a useful phrase or selected practice sample. Verify the receipt.
3. At the end, ask ChatGPT to read saved evidence and draft a report with evidence IDs.
4. Review the draft. Submit it through `save_report`, or paste the same JSON into the private dashboard.
5. The backend checks evidence, ownership, date, and source hash. Changed evidence makes the draft stale.
6. Local, testable rules update review states and prepare the next plan. Unknown evidence stays unknown.

Long evidence lists must be paginated or analyzed in explicit blocks. A report says what was saved, not what happened across all ChatGPT conversations. Imported records follow the same report-validation path.

## Manual export import

Request your official ChatGPT export, download it yourself, and open it in the local importer. Select learning conversations after previewing the archive. The importer must detect actual format, preserve branch choices, deduplicate source IDs and hashes, and reject unsupported structures instead of guessing. Voice and image records remain unknown when no usable text or resource is present.

Local import does not run model analysis. To use the ChatGPT subscription for analysis, intentionally bring selected imported material into a ChatGPT session via a read tool or manual upload, then save the report as above. Do not automatically upload the entire private archive. Protect against ZIP path traversal and excessive expansion, and delete temporary extracted data after import.

## Learning adaptation

Review due items before introducing new expressions. Independent success can lengthen review spacing; prompted answers and copied text do not count as independent recall. Use familiar phrases in new everyday scenes to test transfer. Specific intervals and thresholds remain configurable and must be calibrated using actual practice, not invented success rates.

A2-oriented content is proposed only after repeated independent performance across sessions and scenes, with learner confirmation. It is not a CEFR certificate. New language pairs need their own scripts, normalization, examples, and learning rules.

## Planned features

- [ ] Self-hosted private dashboard and persistent storage.
- [ ] Native/target language preferences and beginner-friendly daily plan.
- [ ] Explicit practice-sample and vocabulary saving, with write receipts.
- [ ] Report drafts generated in ChatGPT and validated on the backend.
- [ ] Phrase reuse, independent-recall checks, and explainable review scheduling.
- [ ] User-selected ChatGPT export import with preview, deduplication, and deletion.
- [ ] Manual JSON input when write tools are unavailable.
- [ ] Account-specific integration tests and documented limitations.

## Roadmap

1. **Pilot:** verify actual account access; complete one text practice > saved evidence > report > next plan loop.
2. **Pre-A1 to A1:** everyday scenes, selected learning records, review and recall. Do not infer productive English ability from Chinese input alone.
3. **A1 to A2:** optional harder tasks after repeated independent performance and learner confirmation.
4. **More language pairs:** configurable language tags, scripts, examples, and learning rules. Native English > Chinese is a later target, not implemented support.
5. **Optional voice and visuals:** only after platform support and data boundaries are tested.

## Limitations and privacy

- MCP is a tool-call interface, not a full-history subscription or a guaranteed per-turn logging stream.
- Model-selected calls may be skipped. A server receipt proves only that submitted record was saved.
- Public submission guidelines prohibit pulling, reconstructing, or inferring the full chat log. Private developer-mode testing avoids public listing review, but is not a blanket exemption from OpenAI terms, usage policies, or privacy obligations.
- Official plan/write and Voice/app descriptions currently disagree. Actual account testing is required.
- Export is manual, account-dependent, and may be incomplete. Import only selected learning conversations; deleted chats cannot be recovered through an export.
- Exported files can contain unrelated private information. Keep parsing local, exclude archives and credentials from Git, and preview before import.
- Reports are based on observed evidence, not CEFR certification, complete coverage, or a measured learning-success guarantee.
- No paid API key is required for the proposed interactive model-analysis path. Other deployment choices, tunnel setup, hosting, or optional API features may require credentials or costs.

## Contributing

Contribute through issues describing the observed account/plan/surface, expected behavior, actual behavior, and a redacted reproduction. Propose contract changes before changing tool inputs. Implementation contributions should include schema tests, authorization tests, idempotency tests, and import regression fixtures. No runnable development commands are supplied because the application has not been implemented.

Useful contributions: account compatibility tests, import fixtures with synthetic data, beginner-learning prompts, schema validation, accessibility, and language-pair packs. Never submit personal exports, ChatGPT cookies, tokens, or production credentials in issues.

## License

**Recommendation, not an adopted license:** MIT for a short permissive license; Apache-2.0 if an explicit patent grant is preferred. Choose one and add its full `LICENSE` file before publishing. No license has been selected by the owner yet.

## Official references

- Developer mode: https://developers.openai.com/api/docs/guides/developer-mode
- Plan and workspace restrictions: https://help.openai.com/en/articles/12584461
- MCP server and tool-call boundaries: https://developers.openai.com/apps-sdk/build/mcp-server
- Public submission and data rules: https://developers.openai.com/apps-sdk/app-submission-guidelines
- Private connectivity: https://developers.openai.com/api/docs/guides/secure-mcp-tunnels
- Data export: https://help.openai.com/en/articles/7260999
- Apps and Voice: https://help.openai.com/en/articles/11487775-apps-in-chatgpt
- ChatGPT Voice: https://help.openai.com/en/articles/20001274-chatgpt-voice
