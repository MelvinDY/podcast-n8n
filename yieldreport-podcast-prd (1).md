# PRD: YieldReport Daily Podcast Automation

**Status:** Draft v1
**Owner:** YieldReport (build partner: [your name])
**Target executor:** Claude Code
**Last updated:** 2026-05-15

---

## 1. Summary

Build an n8n workflow that converts a manually-selected PDF (typically the YieldReport daily publication, but flexible to other source docs) into a ~10-minute MP3 podcast episode. The workflow is human-triggered, runs end-to-end on click, and outputs a date-stamped folder containing the source PDF, generated script, and final audio.

**v1 scope is deliberately narrow:** one PDF in → one MP3 out, single narrator, single voice. No multi-source synthesis, no automated publishing, no transcription/show notes. Those are explicit v2+ concerns.

---

## 2. Goals & non-goals

### Goals

- Reduce per-episode production time from manual scripting + recording (~3-4 hours) to under 15 minutes of human time (PDF selection + review).
- Keep editorial control: a human picks the source PDF and approves the script before audio is generated.
- Produce audio quality indistinguishable from a single-shot ElevenLabs generation, despite necessary chunking.
- Per-episode cost under AUD $5 all-in.

### Non-goals (v1)

- Automated PDF selection or content scraping.
- Multi-host / conversational format.
- Auto-publishing to Spotify/Apple/etc.
- Transcript or show-notes generation.
- Voice cloning of a YieldReport team member (flagged as v2 candidate).
- Music beds, sound design beyond static intro/outro.

---

## 3. User flow

1. Operator selects a PDF (the day's YieldReport publication, or other source).
2. Operator drops the PDF into a watched folder, OR uploads via n8n form trigger.
3. Operator clicks **Start** in n8n.
4. Workflow extracts PDF text → generates script → posts script to Slack for approval.
5. Operator reviews script in Slack, clicks **Approve** (or edits and re-approves).
6. Workflow synthesizes audio via ElevenLabs (chunked + stitched), concatenates with intro/outro, outputs MP3.
7. Workflow notifies operator with link to final MP3.

**Human-in-the-loop checkpoints:** PDF selection (start), script approval (mid). No human step between approval and final MP3.

---

## 4. Technical architecture

### 4.1 Stack

| Component | Choice | Notes |
|---|---|---|
| Orchestration | n8n (self-hosted or cloud) | Manual trigger + linear workflow |
| PDF extraction | n8n `Extract from File` node | Built-in, no external dep |
| Script generation | Google Gemini API (`gemini-2.5-flash`) | Via n8n HTTP Request node or Google Gemini community node |
| Voice synthesis | ElevenLabs `eleven_turbo_v2_5` | Chunked with request stitching |
| Audio concat | FFmpeg via Execute Command node | Stitches intro + body chunks + outro |
| Storage | Google Drive | Dated folders, Drive node for read/write |
| Notifications | Slack | Approval gate + completion notice |

### 4.2 Environment variables (n8n credentials store)

```
GEMINI_API_KEY
ELEVENLABS_API_KEY
ELEVENLABS_VOICE_ID          # selected during voice audition step
GOOGLE_DRIVE_OAUTH           # n8n native credential
SLACK_OAUTH                  # n8n native credential
SLACK_APPROVAL_CHANNEL_ID    # e.g. #podcast-approvals
INTRO_AUDIO_PATH             # /assets/intro.mp3 in n8n filesystem
OUTRO_AUDIO_PATH             # /assets/outro.mp3
OUTPUT_DRIVE_FOLDER_ID       # Google Drive parent folder for episode output
```

### 4.3 Folder structure (Google Drive output)

```
/YieldReport-Podcast/
  /2026-05-15/
    source.pdf              # copy of input PDF
    script.md               # final approved script
    script_chunks.json      # chunked script with request IDs (debug)
    episode.mp3             # final stitched audio
    metadata.json           # duration, voice ID, model, char count, cost estimate
```

---

## 5. n8n workflow specification

Nodes in execution order:

### Node 1: Manual Trigger
- Type: `n8n-nodes-base.manualTrigger`
- No config beyond default.

### Node 2: Read PDF from Drive (or local)
- Type: `n8n-nodes-base.googleDrive` (operation: download) OR `n8n-nodes-base.readBinaryFile`
- Input: PDF file ID or path provided via trigger.
- Output: binary PDF.

### Node 3: Extract PDF text
- Type: `n8n-nodes-base.extractFromFile`
- Operation: `pdf`
- Property to write to: `extractedText`
- Output: `{{ $json.extractedText }}` contains full PDF text.

### Node 4: Generate script (Gemini)
- Type: HTTP Request to `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- Method: POST
- Headers:
  - `x-goog-api-key: {{ $env.GEMINI_API_KEY }}`
  - `Content-Type: application/json`
- Body (JSON):
  ```json
  {
    "contents": [
      {
        "role": "user",
        "parts": [
          { "text": "{{ $node['Build prompt'].json.prompt }}" }
        ]
      }
    ],
    "generationConfig": {
      "temperature": 0.7,
      "maxOutputTokens": 4000,
      "thinkingConfig": {
        "thinkingBudget": 0
      }
    }
  }
  ```
- **Note on `thinkingBudget: 0`:** Gemini 2.5 Flash has reasoning/thinking capability enabled by default. For straightforward script generation from a structured prompt, thinking adds latency and cost without measurable quality gain. Disable it. If output quality drops, raise to a small budget (e.g. 512) before considering Pro.
- **Response parsing:** the script text is at `$json.candidates[0].content.parts[0].text`. Pipe through a Set node to extract it cleanly before downstream nodes.
- See §6 for the prompt itself.

### Node 5: Post script to Slack for approval
- Type: Slack `Send message`
- Channel: `{{ $env.SLACK_APPROVAL_CHANNEL_ID }}`
- Message: script preview + two block-kit buttons: **Approve** | **Reject & edit**
- Pattern: use n8n's `Wait` node with webhook resume — the Slack button click hits the webhook URL to resume the workflow.

### Node 6: Wait for approval (webhook)
- Type: `n8n-nodes-base.wait`
- Mode: webhook
- Resume only on `approved=true` payload.

### Node 7: Chunk script
- Type: Code node (JavaScript)
- Logic: split script on paragraph breaks; greedily pack paragraphs into chunks of max 2,500 chars each; never split mid-sentence; output array `chunks[]`.
- See §7 for the chunking code.

### Node 8: Loop over chunks → ElevenLabs
- Type: `n8n-nodes-base.splitInBatches` (batch size: 1)
- Inside loop: HTTP Request to ElevenLabs (see §8).
- After each call, capture response's `request-id` header into a workflow variable `previousRequestIds[]` (cap at last 3 IDs).
- Save each MP3 chunk to a temp path: `/tmp/chunk_{{ $itemIndex }}.mp3`.

### Node 9: Concatenate audio
- Type: Execute Command
- Command:
  ```bash
  ffmpeg -y -f concat -safe 0 -i /tmp/concat_list.txt -c copy /tmp/episode.mp3
  ```
- `/tmp/concat_list.txt` is built in a preceding Code node listing: intro path, all chunk paths in order, outro path.

### Node 10: Upload to Drive
- Type: Google Drive `Upload`
- Folder: dated subfolder under `{{ $env.OUTPUT_DRIVE_FOLDER_ID }}` (create if missing).
- Upload `source.pdf`, `script.md`, `script_chunks.json`, `episode.mp3`, `metadata.json`.

### Node 11: Notify completion
- Type: Slack `Send message`
- Message: "✅ Episode ready: [link to Drive folder] — duration: Xm Ys, cost: AUD $X.XX"

---

## 6. Script generation prompt

This is the verbatim prompt sent to Claude in Node 4. The `{{ extractedText }}` placeholder is substituted by n8n.

```
You are writing a podcast script for YieldReport's daily fixed income brief.

AUDIENCE
Sophisticated Australian investors: SMSF trustees, financial advisers,
corporate treasurers, fund managers, super fund staff. They know what
a basis point is. They know what hybrids and term deposits are. Do not
explain basics.

TONE
Authoritative, concise, Reuters-meets-Bloomberg. No hype. No filler
phrases ("In this episode we'll explore..."). No conversational
throat-clearing. Lead with substance.

LENGTH
Target 1,500 words. Hard floor 1,300, hard ceiling 1,650. This maps
to roughly 10 minutes of spoken audio at 150 wpm.

STRUCTURE
1. Cold open with the single most important development from the
   source document — one sentence stating what happened.
2. One-paragraph context for that lead story.
3. Sectioned coverage of the relevant areas, only those present in
   the source: cash rates, term deposits, government bonds, corporate
   bonds, hybrids, managed funds. Skip any area the source doesn't
   cover. Don't fabricate coverage.
4. Specific numbers throughout — yields, bps moves, issue sizes,
   coupon rates, maturity dates. Numbers are the value-add.
5. Close with one sentence on what to watch tomorrow or this week.

FORMATTING RULES (CRITICAL — output goes directly to TTS)
- Output only the words the narrator will read. No stage directions.
  No speaker labels. No "[pause]" or "[music]" markers.
- No markdown. No bold. No headers. No bullet points. Plain prose only.
- Spell out numbers that are awkward when read: "twenty-five basis
  points" not "25bps" — but keep precise figures like "4.35%" as
  digits because TTS handles them correctly.
- Write out acronyms on first use: "Reserve Bank of Australia (RBA)"
  then "RBA" thereafter.
- Use full sentences. No sentence fragments. TTS prosody depends on it.
- End every sentence with proper terminal punctuation. No trailing
  commas at section breaks.

DO NOT
- Make up numbers, names, or events not in the source document.
- Recommend buying or selling specific securities. Report; don't advise.
- Use phrases like "as our listeners know" or "stay tuned."

SOURCE DOCUMENT:
{{ extractedText }}

Output the script now. Begin immediately with the cold open. No
preamble, no "Here is the script:" — just the script itself.
```

---

## 7. Chunking logic (Code node, JavaScript)

```javascript
const MAX_CHARS = 2500;
const script = $input.first().json.script;

// Split on paragraph breaks (one or more blank lines)
const paragraphs = script.split(/\n\s*\n/).map(p => p.trim()).filter(Boolean);

const chunks = [];
let current = '';

for (const para of paragraphs) {
  // If adding this paragraph would exceed limit, flush current
  if (current && (current.length + para.length + 2) > MAX_CHARS) {
    chunks.push(current.trim());
    current = '';
  }

  // If a single paragraph exceeds the limit, split on sentence boundaries
  if (para.length > MAX_CHARS) {
    const sentences = para.match(/[^.!?]+[.!?]+/g) || [para];
    for (const sent of sentences) {
      if (current && (current.length + sent.length + 1) > MAX_CHARS) {
        chunks.push(current.trim());
        current = '';
      }
      current += (current ? ' ' : '') + sent.trim();
    }
  } else {
    current += (current ? '\n\n' : '') + para;
  }
}

if (current.trim()) chunks.push(current.trim());

return chunks.map((text, i) => ({ json: { chunkIndex: i, text } }));
```

**Validation:** every chunk must (a) be ≤ 4,500 chars (safety margin under ElevenLabs 5k limit), (b) end with `.`, `?`, or `!`. Add an assertion node that fails loudly if either check fails.

---

## 8. ElevenLabs API call (per chunk)

HTTP Request node config inside the loop:

- **Method:** POST
- **URL:** `https://api.elevenlabs.io/v1/text-to-speech/{{ $env.ELEVENLABS_VOICE_ID }}`
- **Headers:**
  - `xi-api-key: {{ $env.ELEVENLABS_API_KEY }}`
  - `Content-Type: application/json`
  - `Accept: audio/mpeg`
- **Body (JSON):**
  ```json
  {
    "text": "{{ $json.text }}",
    "model_id": "eleven_turbo_v2_5",
    "voice_settings": {
      "stability": 0.55,
      "similarity_boost": 0.75,
      "style": 0.10,
      "use_speaker_boost": true
    },
    "previous_request_ids": {{ JSON.stringify($workflow.previousRequestIds || []) }}
  }
  ```
- **Response handling:** receive as binary, save to `/tmp/chunk_{{ $json.chunkIndex }}.mp3`. Read response header `request-id` and append to `$workflow.previousRequestIds` (trim to last 3).

**Why these settings:**
- `stability: 0.55` — balanced; lower drifts, higher gets monotone.
- `similarity_boost: 0.75` — high voice fidelity.
- `style: 0.10` — minimal style exaggeration; this is a news read.
- `use_speaker_boost: true` — clarity for finance terminology.
- `previous_request_ids` — request stitching, preserves voice consistency across chunks. Critical.

---

## 9. FFmpeg concat (Execute Command node)

Build `concat_list.txt` in a preceding Code node:

```javascript
const lines = [
  `file '${$env.INTRO_AUDIO_PATH}'`,
  ...$('Loop over chunks').all().map((_, i) => `file '/tmp/chunk_${i}.mp3'`),
  `file '${$env.OUTRO_AUDIO_PATH}'`,
];
return [{ json: { concatList: lines.join('\n') } }];
```

Write to disk, then:

```bash
echo "{{ $json.concatList }}" > /tmp/concat_list.txt
ffmpeg -y -f concat -safe 0 -i /tmp/concat_list.txt -c copy /tmp/episode.mp3
```

**Note on `-c copy`:** assumes intro/outro/chunks are all encoded with matching codec/bitrate (MP3, same sample rate). If not, drop `-c copy` and let FFmpeg re-encode — slower but safer. Recommended for v1: re-encode to be safe:

```bash
ffmpeg -y -f concat -safe 0 -i /tmp/concat_list.txt -c:a libmp3lame -b:a 128k /tmp/episode.mp3
```

---

## 10. Pre-build prerequisites

Before the workflow can run end-to-end, the following must exist:

1. **ElevenLabs account** at Creator tier minimum ($22/mo, 100k chars). Pro tier ($99/mo, 500k chars) recommended for daily production. Set up payment.
2. **Voice selection.** Audition at least 4 voices from the ElevenLabs library against a real YieldReport script. Candidates to test: Brian, Daniel, George, Charlotte. Operator listens to ~30 seconds of each reading actual copy. Lock in the chosen `VOICE_ID`.
3. **Intro audio file.** ~10-second pre-recorded clip: "This is the YieldReport daily brief." Store at `INTRO_AUDIO_PATH`. Same codec/bitrate as ElevenLabs output (MP3, 44.1kHz, 128kbps).
4. **Outro audio file.** ~10-second clip: "That's the YieldReport brief. For full data and analysis, visit yieldreport.com.au." Same encoding specs.
5. **Slack approval channel** created, bot invited, webhook URL configured.
6. **Google Drive output folder** created, ID captured in env.
7. **n8n instance** running with sufficient disk for /tmp files (an episode = ~10MB).

---

## 11. Cost model (per episode, AUD)

| Item | Cost |
|---|---|
| Gemini 2.5 Flash (~3k input tokens, ~2k output, no thinking) | ~$0.01 |
| ElevenLabs Turbo v2.5 (~9k chars on Pro tier amortised) | ~$1.80 |
| Storage (Drive) | negligible |
| n8n compute | fixed monthly |
| **Per-episode marginal** | **~$1.85** |

Monthly for 22 weekday episodes: ~$41 marginal + $99 ElevenLabs Pro + ~$20 n8n cloud = **~$160/month**.

---

## 12. Acceptance criteria

The workflow is considered complete and shippable when:

- [ ] Operator can drop a PDF and click Start, with no further input until the Slack approval prompt.
- [ ] Script consistently lands between 1,300–1,650 words across 10 test runs on real YieldReport PDFs.
- [ ] Script approval via Slack button correctly resumes the workflow within 5 seconds of click.
- [ ] Final MP3 duration falls between 8:30 and 11:30 for a 1,500-word script.
- [ ] No audible voice drift, clicks, or seams between chunks in blind A/B test by operator.
- [ ] Final MP3 successfully saves to dated Drive folder with all 5 artefacts present.
- [ ] Slack completion notification fires with accurate duration and cost figures.
- [ ] Workflow handles a 30-page PDF without timeout (validate with longest realistic input).
- [ ] On script generation failure, workflow halts and notifies Slack with error.
- [ ] On ElevenLabs failure mid-chunk, workflow retries chunk up to 2x before halting.

---

## 13. Known risks & mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Voice drift across chunks despite stitching | Medium | Mitigated by `previous_request_ids`; if observed in testing, reduce chunk count by raising MAX_CHARS toward 4,500. |
| Script exceeds word budget on long PDFs | Medium | Prompt has hard ceiling; add post-generation length check + auto-retry with stricter instruction if violated. |
| ElevenLabs rate limit on Pro tier | Low | Pro allows generous concurrent requests; serialise chunks (batch size 1) anyway. |
| Compliance: script mentions specific securities | High | Mandatory human approval gate before audio synthesis. Operator is the compliance backstop in v1. |
| FFmpeg codec mismatch causes concat failure | Medium | Use re-encode variant (§9) rather than `-c copy` in v1. |
| Source PDF is image-only (scanned) | Low | Add OCR fallback (Tesseract via Execute Command) in v2; in v1, fail loudly with a clear Slack error. |

---

## 14. v2+ candidates (not in v1 scope)

- Voice cloning of a YieldReport spokesperson (one-time setup, Professional Voice Cloning tier).
- Auto-publish to Spotify/Apple via Transistor or Captivate API.
- Auto-generate show notes + timestamps from the script.
- Multi-source synthesis: combine YieldReport PDF + RBA statement + market data into one episode.
- Scheduled trigger replacing manual click, once editorial trusts the output.
- Two-host conversational format using two distinct voice IDs and a dialogue-formatted script.
- Web dashboard for non-technical operators (replacing n8n's manual trigger UI).

---

## 15. Build sequence for Claude Code

Suggested order of implementation:

1. Scaffold n8n workflow with Manual Trigger → Read PDF → Extract → static script output. Verify text extraction quality.
2. Add Gemini call with the §6 prompt. Tune until 5 consecutive test PDFs produce in-spec scripts. Verify response parsing path (`candidates[0].content.parts[0].text`) handles edge cases.
3. Add Slack approval gate. Verify webhook resume works.
4. Add chunking Code node. Unit-test against 10 sample scripts; verify all chunks pass validation.
5. Add ElevenLabs loop with stitching. Confirm `request-id` is captured from response headers.
6. Add FFmpeg concat. Verify with intro + 3 chunks + outro produces a clean MP3.
7. Add Drive upload + metadata. Verify folder structure matches §4.3.
8. Add completion Slack notification.
9. Run end-to-end on 5 real YieldReport PDFs. Iterate on prompt/settings against acceptance criteria (§12).

---

## Appendix A: Open questions for stakeholders

1. Who owns the approval step in Slack? Single approver or any team member?
2. Episodes published weekdays only, or 7-day cadence?
3. Voice cloning candidate — willing to record 30 mins of clean audio for v2?
4. Compliance review needed by anyone outside the operator?
5. Existing intro/outro assets, or do these need to be produced?
