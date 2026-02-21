# OpenClaw Voice Interop Plan for `@fluxerjs/voice`

This plan focuses on one gap blocking first-class OpenClaw voice support on Fluxer:

- ✅ playback to LiveKit voice channels already works
- ❌ receiving remote participant audio for transcription is not exposed yet

The goal is to add a receive pipeline that feels native to `@fluxerjs/voice` and aligns with the existing code style and contribution workflow.

---

## 1) Current State (Repo Reality)

### Working now

- `VoiceManager.join(channel)` negotiates voice and returns either:
  - `VoiceConnection` (Discord-style UDP path)
  - `LiveKitRtcConnection` (LiveKit path)
- `LiveKitRtcConnection.play(...)` publishes bot audio into room
- `LiveKitRtcConnection.playVideo(...)` supports camera/screenshare publishing

### Missing now

- No public API to subscribe to remote audio tracks
- No speaking lifecycle events suitable for STT pipelines
- `Room.connect(..., { autoSubscribe: false })` explicitly avoids receiving tracks

### Relevant files

- `packages/voice/src/LiveKitRtcConnection.ts`
- `packages/voice/src/VoiceManager.ts`
- `packages/voice/src/index.ts`
- `packages/voice/README.md`

---

## 2) Product Goal

Enable downstream apps (OpenClaw included) to:

1. detect who is speaking in a voice room
2. subscribe to a participant’s inbound audio stream
3. consume decoded PCM frames (or equivalent frame abstraction)
4. unsubscribe cleanly when participant stops/leaves

This should support **near-real-time transcription** and be resilient across reconnects.

---

## 3) API Design (Proposed)

Add LiveKit receive APIs without breaking existing users.

### New types

```ts
export type LiveKitAudioFrame = {
  participantId: string;
  trackSid?: string;
  sampleRate: number; // 48000
  channels: number;   // 1 for now
  samples: Int16Array;
  timestampUs?: bigint;
};
```

```ts
export type LiveKitReceiveSubscription = {
  participantId: string;
  stop: () => void;
};
```

### New `LiveKitRtcConnection` methods

```ts
subscribeParticipantAudio(participantId: string, options?: { autoResubscribe?: boolean }): LiveKitReceiveSubscription;
on('audioFrame', (frame: LiveKitAudioFrame) => void): this;
on('speakerStart', ({ participantId }) => void): this;
on('speakerStop', ({ participantId }) => void): this;
```

### Behavior notes

- Keep `play()` and `playVideo()` unchanged
- Receive API should be opt-in (don’t decode everyone by default)
- If participant has multiple audio tracks, pick first microphone source by default
- Auto-cleanup on disconnect/reconnect and participant leave

---

## 4) Implementation Plan (Phased)

## Phase A — Receive scaffolding in `LiveKitRtcConnection`

**Files**
- `packages/voice/src/LiveKitRtcConnection.ts`

**Changes**
- Add internal subscription registry keyed by `participantId`
- Add room event listeners:
  - `RoomEvent.TrackSubscribed`
  - `RoomEvent.TrackUnsubscribed`
  - `RoomEvent.ParticipantDisconnected`
- Build per-participant audio pipeline (LiveKit audio stream -> int16 frames)
- Emit new typed events: `audioFrame`, `speakerStart`, `speakerStop`
- Ensure `disconnect()/destroy()/stop()` clears receive resources

**Style alignment**
- Follow existing private field and event-emitter patterns used in this file
- Keep debug logging behind existing debug helper conventions
- Avoid introducing alternate logging frameworks

## Phase B — VoiceManager integration helpers

**Files**
- `packages/voice/src/VoiceManager.ts`

**Changes**
- Add helpers for current voice-state map reuse:
  - `listParticipantsInChannel(guildId, channelId): string[]`
- Optional convenience wrapper:
  - `subscribeChannelParticipants(channelId, opts?)`
- Keep existing join/leave semantics untouched

## Phase C — Public exports + docs

**Files**
- `packages/voice/src/index.ts`
- `packages/voice/README.md`

**Changes**
- Export new receive types/events
- Add “Inbound transcription” example snippet showing per-participant subscribe
- Document lifecycle and cleanup expectations

## Phase D — Test coverage

Given current package lacks focused tests, add lightweight targeted tests first.

**Suggested test files**
- `packages/voice/src/__tests__/LiveKitRtcConnection.receive.test.ts`
- `packages/voice/src/__tests__/VoiceManager.receive.test.ts`

**Test scope**
- event emission correctness (`speakerStart/Stop`, `audioFrame`)
- unsubscribe/cleanup behavior
- reconnect path does not leak handlers
- no crash when track disappears mid-stream

If full LiveKit integration tests are heavy for CI, unit-test with mocked room/participant/track surfaces.

---

## 5) OpenClaw Mapping (Why this is enough)

Once receive exists, OpenClaw can implement Fluxer voice with near-parity to Discord:

1. join channel via `VoiceManager.join(...)`
2. subscribe to participant audio frames
3. write WAV chunks or direct PCM handoff to transcription
4. call agent runtime
5. synthesize TTS
6. play back with existing `play(...)`

No protocol-level LiveKit work needed in OpenClaw itself; `@fluxerjs/voice` remains the transport adapter.

---

## 6) Risks + Mitigations

### Risk: CPU overhead decoding many participants
- **Mitigation:** opt-in per-participant subscription; avoid global auto-subscribe decode

### Risk: frame timing jitter under load
- **Mitigation:** bounded queues + backpressure + frame drop policy for stale buffers

### Risk: reconnect leaks
- **Mitigation:** central teardown for all receive subscriptions in `disconnect()` and room `Disconnected` event

### Risk: API lock-in too early
- **Mitigation:** keep v1 API minimal (`audioFrame` + subscribe/unsubscribe), defer advanced knobs

---

## 7) Contribution Workflow (per repo conventions)

From `CONTRIBUTING.md` and existing project patterns:

1. Branch from `main`
2. Use Conventional Commits (`feat:` for receive API)
3. Add changeset for `@fluxerjs/voice`
4. Run:
   - `pnpm run lint`
   - `pnpm run build`
   - `pnpm run test`
5. Open PR with clear checklist completion

Suggested commit sequence:

1. `feat(voice): add livekit participant audio receive subscriptions`
2. `feat(voice): add voice manager helpers for channel participant subscriptions`
3. `docs(voice): document inbound transcription receive pipeline`
4. `test(voice): cover receive lifecycle and cleanup`
5. `chore(changeset): bump @fluxerjs/voice for receive APIs`

---

## 8) Proposed Minimal Deliverable (MVP)

If we want quickest merge path:

- implement **Phase A + C only**
- leave `VoiceManager` convenience helpers for follow-up PR

This provides immediate value for OpenClaw while keeping first PR narrowly scoped.

---

## 9) Nice-to-Have Follow-ups

- VAD/speaking threshold tuning events
- mixed-audio stream option (channel-level merge)
- direct WAV/PCM utilities bundled in package
- metrics hooks for packet loss and jitter stats

---

## 10) Definition of Done

- Downstream can subscribe to participant audio and receive stable PCM frames
- No regressions to existing `play()` / `playVideo()` behavior
- reconnect/disconnect cleanup verified
- docs + changeset included
- CI green
