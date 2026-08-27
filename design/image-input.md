# Image input

## Status

Implemented.

## Problem

Artemis submitted Discord messages to the model as text only. Image attachments
in a triggering message were dropped silently, so users could not share
screenshots, diagrams, or photos with a vision-capable model, and deployments
running text-only models had no way to state that boundary through
configuration.

## Scope

This protocol owns:

- the optional `supportsImageInput` provider capability flag
- Discord image-attachment collection, validation, and download
- image-content delivery to the PI prompt boundary
- prompt annotation for attached images

It does not change Discord authorization, conversation identity, tool
registration, response delivery, or the normalized SQLite transcript schema.
Thread-history images other than the triggering message's attachments are not
re-submitted; thread snapshots keep their text-only contract.

## Observable behavior

With the default configuration the behavior is unchanged: image attachments are
ignored and image-only messages with no text are treated as empty.

When the configured model provider declares `supportsImageInput`, Artemis
collects the triggering message's image attachments after authorization and
deduplication, downloads them, and submits them to the model as image content
alongside the existing JSON message prompt. The triggering message may now
consist of images with no text. Non-image attachments are still ignored. The
`discord_message_received` audit event gains an `imageAttachments` summary
(attachment ID, content type, declared size) whenever a received message
carries image attachments, before any download happens.

## Contracts and data flow

`MODEL_CONFIG_PATH` (or the legacy Ollama defaults) may declare
`supportsImageInput: boolean`, defaulting to `false`. When `true`, the PI model
registration advertises `input: ["text", "image"]`; PI forwards image content
to the provider only when the registered model declares image input. A
text-only model with the flag enabled fails generation at the provider, so the
flag must only be set for providers that accept image content.

The runtime flow is:

```text
Discord message -> audit image-attachment metadata (synchronous)
  -> accepted turn -> download attachments (image/*) -> validate type and size
  -> base64 -> ConversationService -> PiGenerationInput.images
  -> PI session.prompt(text, { images }) -> OpenAI-compatible content parts
```

Attachment download runs lazily inside the generation path, after Discord
authorization and duplicate-message deduplication, so ignored messages never
fetch attachment bytes. Intake limits are fixed in application code:

- accepted content types: `image/png`, `image/jpeg`, `image/webp`, `image/gif`
- at most 4 image attachments per triggering message
- at most 10 MiB per image after download

A message with collected images keeps the existing prompt format and appends a
deterministic note stating how many image attachments accompany the newest
Discord message. The note is appended for both single-message and thread
snapshot prompts; images from thread history are not re-attached.

## Configuration

| Setting | Default | Purpose |
| --- | --- | --- |
| `supportsImageInput` (model config JSON) | `false` | Enables image-attachment collection and advertises image input to PI. |

The flag is a property of the configured provider, not a separate environment
variable. Deployments that run vision-capable models (for example a
GLM-5.3-Flash endpoint) set it in `model.config.json`; the Ollama default and
existing provider files omit it and keep the previous behavior.

## Persistence

No SQLite schema changes. Attachment bytes are not stored in the normalized
transcript; `incoming_messages` and `messages` keep text content only. Image
content lives inside the native PI session entries that record each turn's user
message, so restored sessions keep their image context without a new table.
Attachment metadata (ID, content type, declared size) is logged in the
`discord_message_received` audit event for operator correlation; downloaded
bytes exist only for the duration of a turn.

## Security and privacy

Image attachments are untrusted external content delivered directly to the
model. Unlike `web_fetch` and GitHub tool text, images are not sanitized: the
model may act on whatever the image depicts, including text rendered inside the
image. Authorization still gates who can trigger a turn, and the size and
content-type limits bound resource use. Images are fetched from Discord's CDN
at ingestion time because attachment URLs are signed and expire.

## Failure handling

- An image attachment with an unsupported `image/*` content type, too many
  image attachments, an oversized image, or a failed download fails the turn
  with a descriptive error, following the normal generation-failure path
  (logged, persisted, no Discord response). Partial contexts are never
  submitted silently.
- A non-`2xx` provider response for image content follows the existing
  PI/model-provider failure path.

## Verification

- `test/config.test.ts` covers the `supportsImageInput` default, explicit
  values, and validation.
- `test/discord-gateway.test.ts` covers collection gating, content-type and
  size limits, download failure, non-image filtering, and audit metadata.
- `test/conversation-service.test.ts` covers image passthrough, the empty-text
  guard, and the prompt annotation.
- `test/pi-gateway.test.ts` covers PI model registration input advertising and
  image options passed to `session.prompt`.
- `npm run guardrail` remains the completion gate.

## References

- [Baseline design](baseline.md)
- [Configurable model provider](model-provider.md)
- [Clean-room rebuild guide](rebuild-guide.md)
- [Design document index](README.md)
