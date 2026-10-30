
# AI HR Helper

Recruitment assistant with two parts: a CV intake form that parses and scores every resume, and a Telegram bot where HR staff search the candidate database in Egyptian Arabic.

Built for the Egyptian market (Arabic replies for HR staff, local phone formats preserved as written in the CV).

## Features

**CV intake (web form)**
- Public form with a single file upload field
- Stores the original CV in a Google Drive folder and keeps its share link
- Parses the document with LlamaParse, then an LLM extracts structured data: contact details, education, experience, skills (technical, soft, languages, frameworks, databases, cloud, tools), certifications, projects, awards, courses
- Extracts only what the CV states, with `null` for missing fields (no guessing)
- Scores the CV (0 to 100), sets an experience level, lists strengths, weaknesses, and missing information, and gives an HR recommendation
- Appends a candidate row to Google Sheets with name, email, phone, and CV link

**HR Telegram bot**
- Search candidates by name, phone, email, skills, job title, or city
- Summarize and compare candidates, list top candidates
- Answers only from the sheet, never invents candidates, replies short in Egyptian Arabic
- Per-chat memory (last 10 messages)

## Tech stack

| Layer | Tools |
|---|---|
| Automation | n8n (form trigger, Telegram trigger, AI Agent, LLM chain, structured parser) |
| Document parsing | LlamaParse (cost-effective tier) |
| LLM | OpenRouter `openai/gpt-oss-20b:free` (parser chain and agent) |
| Storage | Google Drive (CV files), Google Sheets (candidate table) |
| Channel | Telegram Bot API |

## Architecture

```
HR / candidate ─► Form (cv file) ─► Drive upload ─► link + id ─► Drive download
                                                                       │
                                                                       ▼
                                                              LlamaParse (text)
                                                                       │
                                                                       ▼
                                                  LLM chain + structured output parser
                                                  (extract, score, recommend)
                                                                       │
                                                                       ▼
                                                          Google Sheets (append row)
                                                                       ▲
Telegram ─► AI Agent ◄── memory (10) ──────────────────────────────────┘
               │   ◄── tool: get rows from sheet
               ▼
        chat action ─► wait 3s ─► reply
```

## Getting started

### 1. Import

Import the workflow JSON into n8n and re-link every credential.

### 2. Credentials

| Credential | Type | Used by |
|---|---|---|
| Google Drive | `googleDriveOAuth2Api` | Upload and download CVs |
| LlamaParse | `llamaParseApi` | CV text extraction |
| OpenRouter | `openRouterApi` | Parser chain and agent |
| Google Sheets | `googleSheetsOAuth2Api` | Append rows, agent search tool |
| Telegram | `telegramApi` | Trigger and replies |

### 3. Drive and Sheet

1. Create a Drive folder for CVs and select it in the `Upload file` node.
2. Create a sheet with the columns you want to store. Currently mapped: `fullname`, `email`, `phone`, `cv`. See Known issues before going live.

### 4. Run

1. Activate the workflow and open the form production URL.
2. Upload a CV, then check Drive and the sheet.
3. Message the Telegram bot, for example "مين عنده خبرة في Python؟".

## Known issues

| Issue | Fix |
|---|---|
| **Prompt and parser disagree.** The prompt asks for keys like `phone`, `experience_level`, `hr_recommendation`, nested `education` and `experience` arrays. The output parser requires `phone_number`, `location`, `job_title`, `skills`, `experience_years`, `cv_rating`, `evaluation`, and a string `education`. Validation will fail or return the wrong shape | Pick one schema, put it in the parser, and make the prompt reference the same keys |
| **The sheet stores only name, email, phone, and CV link**, but the agent prompt promises search by skills, job title, city, experience, and CV score. Those questions cannot be answered | Add columns (`job_title`, `skills`, `city`, `experience_years`, `cv_score`, `recommendation`) and map them in `Append row in sheet` |
| Sheet column is named ` fullname` with a leading space | Rename the header and the mapping |
| Agent tool reads the entire sheet on every question, no filters. It breaks on token limits as the table grows | Move to Supabase or Postgres with a filtered query tool, or add lookup filters via `$fromAI` |
| Typing indicator (`Send a chat action`) runs after the agent has already answered, then waits 3 s | Send the chat action before the agent and drop the wait |
| CV is uploaded to Drive and then downloaded again, although the binary is already in the workflow | Pass the form binary straight to LlamaParse |
| No duplicate check, the same candidate applying twice adds two rows | Look up by email and update instead of append |
| Form accepts any file type | Restrict accepted extensions to PDF, DOC, DOCX |
| Both models are a free tier model, rate limited and weak at long structured output | Use a paid model for the parser |
| `$json.text` is assumed to be the LlamaParse output field | Check the node output and map the right field |

## Security notes

- **Add auth to the Telegram bot.** Anyone who finds it can read candidate phones and emails. Allowlist HR chat IDs and ignore everyone else.
- CVs contain personal data. Free OpenRouter models may log or train on prompts, so use a provider with a no-retention policy and a proper data processing agreement before real candidates go through it.
- The pipeline extracts gender, age, nationality, and date of birth, and the LLM produces a hiring recommendation. Remove protected attributes from scoring, and treat the score as a hint for a human reviewer, never an automatic decision. Automated employment screening is regulated in some jurisdictions.
- The form URL is public. Add rate limiting or a shared secret, and protect the Drive folder with least-privilege access.
- Keep all keys in n8n credentials. Never inline them in nodes or prompts.
- Do not commit exported workflow JSON with credential IDs, folder IDs, or sheet IDs to a public repo.

## Project structure

```
workflow.json     exported n8n workflow
README.md
```

## Author

**Faris Ataa Lashen**, AI Automation Engineer (n8n, AI Agents, RAG, LLMs)
Portfolio: https://farislashen7-hub.github.io/portfolio · LinkedIn: https://linkedin.com/in/farislashen
