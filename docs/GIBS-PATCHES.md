# GIBS patches on top of upstream OpenWA

This fork (`comitargentina/OpenWA`) carries a small, deliberate patch set on top of `rmyndharis/OpenWA`. The deployed
bridge is **an upstream release tag + the patch branches merged on top**, never a drifting branch. Branches are named
`gibs/<what>`; each is based on the upstream release tag it was written against and is merged forward by the GIBS
updater (`ops/bridge-gateway/auto-update/openwa-update.sh` in the GIBS repo).

Rules for patches: add-only where possible, no edits to upstream's generated or counted files (`openapi.json`,
`docs/06`, `docs/29` counters, `CHANGELOG.md`), so a merge of the next upstream release conflicts as little as it can.
Upstream's own docs-consistency tests (`npm run test:docs`) therefore fail on a patched tree; they guard upstream's
docs, not our behaviour. The tests that matter for a patch are the unit specs of the touched modules.

## gibs/poll-votes — read a poll's votes

`GET /api/sessions/{sessionId}/messages/{chatId}/{messageId}/poll-votes`

Answers `{ "options": ["A", "B"], "votes": [{ "voter": "<wa id, may be @lid>", "options": ["A"], "interactedAtTs": 1791173400 }] }`.
`votes` is every voter's CURRENT selection (an empty `options` array means the voter cleared it) and is `[]` while nobody
has voted. `interactedAtTs` is in MILLISECONDS (verified live on whatsapp-web.js 1.34.7). `messageId` is the poll creation message id returned by `send-poll`. The poll must be inside the same
100-message window the engine fetches for the chat (404 otherwise); a message that is not a poll is a 400; Baileys answers
501. Implemented on whatsapp-web.js with `Message.getPollVotes()` + `Message.pollOptions`; `interactedAtTs` is passed
through exactly as the library reports it.

Files: `engine/interfaces/whatsapp-engine.interface.ts` (types + `getPollVotes`), `engine/adapters/wwebjs-messaging.ts`,
`engine/adapters/whatsapp-web-js.adapter.ts`, `engine/adapters/baileys.adapter.ts` (501), `engine/engine-capability-matrix.ts`
(curated row), `modules/message/message.service.ts`, `message.controller.ts`, `dto/message-responses.dto.ts`, and specs.

### Webhook `message.poll_vote` (same branch)

Upstream has no poll-vote event. whatsapp-web.js already emits `vote_update`; this patch forwards it as the webhook event
`message.poll_vote` (the way `call.received` is forwarded: engine callback `onPollVote` -> `webhookService.dispatch`).
Payload: `{ sessionId, pollMessageId, chatId, voter, options, interactedAtTs }` where `pollMessageId` is the id `send-poll`
returned, `options` is the voter's CURRENT selection by option text (empty = cleared), `interactedAtTs` is milliseconds.
One event per change: a voter changing A -> B yields two events (idempotency keys are salted per occurrence). A webhook
must list `message.poll_vote` in its `events` to receive it. Baileys never fires it. Extra files: `wwebjs-message-events.ts`,
`session-engine-event-wiring.ts`, `webhook.dto.ts` (event catalogue), `webhook/utils/idempotency.util.ts`, and specs.
