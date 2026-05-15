# YieldReport Podcast Automation — n8n Workflow

Converts a manually-selected YieldReport PDF into a ~10-minute MP3 podcast episode. One PDF in, one MP3 out. Human-in-the-loop at two points: PDF selection and script approval.

## What it does

1. Operator opens the n8n form and pastes a Google Drive File ID
2. Workflow downloads the PDF, extracts text, generates a 1,300–1,650-word script via Gemini 2.5 Flash
3. Script is posted to Slack with Approve / Reject buttons
4. On approval: chunks script, synthesizes audio via ElevenLabs (with request stitching for voice consistency), concatenates with intro/outro via FFmpeg
5. Uploads all artefacts to a dated Google Drive folder
6. Posts completion notice to Slack with Drive link, duration, and cost estimate

## Prerequisites

Before the workflow can run end-to-end:

| Item | Notes |
|---|---|
| n8n instance | Self-hosted or cloud. Needs FFmpeg installed (`apt install ffmpeg`). |
| Google Drive OAuth credential | Set up in n8n credentials store. |
| Slack bot | Create at api.slack.com/apps. Scopes: `chat:write`, `chat:write.public`. |
| ElevenLabs account | Creator tier minimum ($22/mo, 100k chars). Pro ($99/mo) recommended for daily use. |
| Intro audio | ~10s MP3 at 44.1kHz 128kbps: "This is the YieldReport daily brief." |
| Outro audio | ~10s MP3 same spec: "That's the YieldReport brief. Visit yieldreport.com.au." |
| Google Drive output folder | Create the parent `/YieldReport-Podcast/` folder and copy its ID. |

## Setup

### 1. Configure environment variables in n8n

Go to **n8n Settings → Variables** and add every key from `.env.example`:

```
GEMINI_API_KEY
ELEVENLABS_API_KEY
ELEVENLABS_VOICE_ID
SLACK_BOT_TOKEN
SLACK_APPROVAL_CHANNEL_ID
OUTPUT_DRIVE_FOLDER_ID
INTRO_AUDIO_PATH
OUTRO_AUDIO_PATH
```

### 2. Add Google Drive OAuth credential

In n8n, go to **Credentials → New → Google Drive OAuth2 API**. Complete the OAuth flow. Note the credential ID — you'll need to replace `REPLACE_WITH_CREDENTIAL_ID` in the workflow JSON before importing.

### 3. Update credential IDs in workflow.json

Search `workflow.json` for `REPLACE_WITH_CREDENTIAL_ID` and replace with your actual Google Drive credential ID. There are 5 occurrences (one per Google Drive node).

### 4. Import the workflow

In n8n, go to **Workflows → Import from File** and select `workflow.json`.

### 5. Place audio assets on the n8n server

Copy `intro.mp3` and `outro.mp3` to the paths you set in `INTRO_AUDIO_PATH` and `OUTRO_AUDIO_PATH`. On Docker: mount a volume or copy files into the container.

### 6. Activate the workflow

Toggle the workflow to active. The form trigger will become available.

## Running an episode

1. Open the form trigger URL (shown in n8n when workflow is active)
2. Paste the Google Drive file ID of the source PDF
3. Optionally set the episode date (defaults to today)
4. Click Submit
5. Wait for the Slack approval message (~30–60s for Gemini)
6. Review the script preview, click **Approve** or **Reject**
7. On approval: ElevenLabs synthesis + FFmpeg concat runs (~5–10 min for a full episode)
8. Slack completion notice arrives with Drive folder link and cost

## Output folder structure

```
/YieldReport-Podcast/
  /2026-05-15/
    source.pdf          ← copy of input PDF
    script.md           ← approved script
    script_chunks.json  ← chunked script debug info
    episode.mp3         ← final stitched audio
    metadata.json       ← duration, cost, voice ID, model, word count
```

## Cost model (per episode, AUD)

| Item | Cost |
|---|---|
| Gemini 2.5 Flash (~3k input, ~2k output tokens) | ~$0.01 |
| ElevenLabs Turbo v2.5 (~9k chars on Pro tier) | ~$1.80 |
| **Total marginal per episode** | **~$1.85** |

## ElevenLabs voice settings

Configured for a news/finance read:

| Setting | Value | Reason |
|---|---|---|
| `stability` | 0.55 | Balanced — lower drifts, higher gets monotone |
| `similarity_boost` | 0.75 | High voice fidelity across chunks |
| `style` | 0.10 | Minimal style exaggeration |
| `use_speaker_boost` | true | Clarity for financial terminology |
| `previous_request_ids` | last 3 | Request stitching — preserves voice consistency across chunks |

## Troubleshooting

**PDF extraction returns too little text**
The source PDF may be image-only (scanned). v1 does not support OCR — the workflow will fail loudly with a clear error message. Use a text-based PDF.

**Script word count validation fails**
If Gemini returns a script outside 1,300–1,650 words, the workflow halts before posting to Slack. Check the Gemini response in the n8n execution log. Do not modify the prompt without reviewing the PRD §6.

**FFmpeg codec mismatch**
The workflow uses `-c:a libmp3lame -b:a 128k` (re-encode) rather than `-c copy` for v1 safety. Ensure intro/outro assets are valid MP3 files.

**Slack buttons don't resume the workflow**
URL-type Slack buttons open the resume URL in the browser. If the browser shows an error, check that the n8n instance is publicly accessible and the workflow is active.

**ElevenLabs `request-id` not found in headers**
The workflow looks for `request-id` and `x-request-id` headers. If neither is present, request stitching will still work (previous_request_ids will just be empty), but voice consistency across chunks may degrade.
