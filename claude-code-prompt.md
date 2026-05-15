# Claude Code Prompt: Build YieldReport Podcast Workflow

Copy everything below the divider into Claude Code.

---

You are building an n8n workflow per the PRD at `yieldreport-podcast-prd.md` (read it in full before doing anything else). You have the n8n MCP server installed and should use it as your primary tool for this build — do not write workflow JSON by hand when the MCP can introspect, create, and validate nodes for you.

## Operating principles

1. **PRD is the source of truth.** Read it end-to-end first. If anything in this prompt contradicts the PRD, the PRD wins. If anything is genuinely ambiguous in both, ask me before guessing.
2. **Use the MCP, don't reinvent.** Before creating any node, use the MCP's tools to look up the node type, its current parameter schema, and any built-in operations. n8n's node API changes; your training data is not authoritative. The MCP is.
3. **Build incrementally, test at each stage.** Do not build all 11 nodes then run end-to-end. Build → test → build → test, following the sequence in PRD §15. Each stage should produce a runnable workflow before you move on.
4. **Show me what you're doing.** After each stage, post a short summary: what you built, what you tested, what the output was, what's next. Don't silently chain 20 tool calls.
5. **Surface decisions, don't bury them.** If the MCP returns multiple node options for a task (e.g. two ways to call HTTP), tell me what you chose and why in one sentence.

## Before you start

Run these in order and report back:

1. **Discover the MCP surface.** List the n8n MCP tools available to you. Confirm you can: list node types, get node schemas, create workflows, add/configure nodes, list credentials, execute workflows for testing.
2. **Check the n8n instance.** Confirm you can reach the n8n instance via MCP. Report the n8n version. Flag if anything looks wrong (auth, version mismatch, etc.).
3. **Check credentials.** List existing credentials. Tell me which of the PRD §4.2 env vars / credentials are already configured and which I need to set up before you can proceed:
   - `GEMINI_API_KEY`
   - `ELEVENLABS_API_KEY` + `ELEVENLABS_VOICE_ID`
   - Google Drive OAuth
   - Slack OAuth + approval channel ID
   - Intro/outro audio file paths
   - Output Drive folder ID
4. **Stop and wait for me to confirm credentials are ready** before creating any nodes. Do not stub them. Do not proceed past this checkpoint without explicit go-ahead.

## Build sequence

Follow PRD §15 verbatim. After each numbered step, **pause and report**:

- What you built (node names, types, key params).
- What you tested and the result.
- Any deviations from the PRD and why (e.g. the MCP exposed a better node for this).
- What's next.

Wait for my "continue" before moving to the next step. The only exception: if a step is trivially small (e.g. "rename a node"), batch it into the next substantive step.

## Specific guidance per stage

**Stage 1 — Scaffold + PDF extract.** Use a real test PDF. I'll provide one, or use any YieldReport PDF in the Drive folder. Verify the extracted text is clean (no PDF artefacts, page numbers stripped reasonably, no encoding garbage). If extraction quality is poor on a real YieldReport PDF, flag it before proceeding — we may need OCR fallback even in v1.

**Stage 2 — Gemini script generation.** The prompt in PRD §6 is verbatim — do not paraphrase or "improve" it. Place it in a Code or Set node that constructs the final prompt by substituting `{{ extractedText }}`. Validate the response parsing handles the Gemini shape (`candidates[0].content.parts[0].text`). Run 5 test PDFs and report:

- Word count of each output (target 1,300–1,650).
- Whether each script ends with proper terminal punctuation.
- Whether any script contains markdown, stage directions, or speaker labels (it shouldn't).
- Whether `thinkingBudget: 0` is actually being honoured (check token usage in the response).

If word count is consistently off, do not change the prompt — report back and we'll tune together.

**Stage 3 — Slack approval gate.** Use n8n's Wait node with webhook resume. Build the Slack message with Block Kit (two buttons). Test the webhook resume path with a real button click. Confirm the workflow correctly distinguishes Approve from Reject.

**Stage 4 — Chunking.** Implement the JavaScript from PRD §7 in a Code node exactly as written. Then write a validation Code node that fails loudly if any chunk exceeds 4,500 chars or doesn't end with `.`, `?`, or `!`. Unit-test against the 5 scripts from Stage 2.

**Stage 5 — ElevenLabs loop with stitching.** The `previous_request_ids` mechanism is the key thing to get right. The `request-id` comes back in the **response header**, not the body — make sure the HTTP node is configured to expose response headers. Maintain the rolling array of last-3 IDs across loop iterations. Save each chunk to `/tmp/chunk_{index}.mp3`. After the loop, report: chunk count, total chars sent, ElevenLabs char usage from response metadata.

**Stage 6 — FFmpeg concat.** Use the re-encode variant from PRD §9 (not `-c copy`) for v1 safety. Verify the output MP3 plays cleanly end-to-end. Listen for: clicks at chunk boundaries, voice drift between sections, intro/outro level mismatch. Report duration vs. expected (≈10 min for a 1,500-word script).

**Stage 7 — Drive upload + metadata.** Folder structure per PRD §4.3 exactly. Create the dated subfolder if it doesn't exist. `metadata.json` should include: timestamp, source PDF filename, script word count, chunk count, total ElevenLabs chars, voice ID used, model used, total duration in seconds, estimated cost in AUD.

**Stage 8 — Completion Slack notification.** Include the Drive folder link, duration, and cost from metadata.json.

**Stage 9 — End-to-end on 5 real PDFs.** Run the full workflow on 5 different real YieldReport PDFs back to back. After each run, report the PRD §12 acceptance criteria status. Flag any criterion that failed.

## Things to ask me about, not guess

- Which Slack channel to use for approvals (if multiple candidates exist).
- Which Google Drive folder is the output root.
- Whether to use Gemini's free tier or a paid key for development.
- Which ElevenLabs voice ID to use (this should be locked in before you start — if it isn't, stop and ask).
- Whether intro/outro audio files exist; if not, we need them before Stage 6.

## Things you should just decide

- Node naming conventions inside the workflow (use clear, consistent names).
- Whether to split a step into multiple sub-nodes for clarity.
- Error handling specifics (retry counts, timeout values) — use sensible defaults, document them.
- Whether to use the n8n native Google Gemini node (if one exists in the MCP listing) versus the raw HTTP Request node. If a native node exists and works, prefer it.

## Done definition

You are done when PRD §12 acceptance criteria all pass on 5 consecutive real-PDF runs, and the workflow is saved in n8n with a clear name like `yieldreport-podcast-v1`.

Now: start with the "Before you start" checks and stop at the credentials checkpoint. Report back.
