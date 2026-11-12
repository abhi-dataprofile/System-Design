# Real-time multimodal agents, voice streaming, and interruption handling

A text chatbot can wait for a complete request. A live voice or video agent cannot: it receives partial observations while producing output that the user may interrupt. The design problem is therefore not merely “stream tokens faster.” It is maintaining a truthful, cancellable session while audio, video, model output, and tools advance on different clocks.

## Mental model: a live session is an event stream, not a request

Model the system as a bidirectional session:

```text
capture -> transport -> jitter buffer -> turn detection -> recognition/model
                                                     -> tools
playback <- transport <- audio generation <- response planning
```

Keep five states distinct:

1. **Tentative:** partial transcript, provisional speaker label, sampled video frame, or unfinished model plan.
2. **Committed:** an observation accepted as part of the conversation record.
3. **Generated:** text or audio produced by the agent but not necessarily sent.
4. **Delivered:** bytes acknowledged by the client.
5. **Heard:** the playback prefix the client reports as actually played.

Generated does not mean delivered; delivered does not mean heard; heard does not mean authorized. This distinction is what makes interruption and recovery correct.

Give every event a session ID, monotonically increasing sequence number, media timestamp, producer, and stability level. The durable conversation record should contain committed events and tool outcomes—not every unstable ASR hypothesis or audio frame.

## Start with a latency budget

Measure each stage instead of hiding all delay behind “model latency”:

[
T_{response}=T_{capture}+T_{network}+T_{jitter}+T_{endpoint}+T_{model-first}+T_{speech-first}+T_{playback}
]

Track distributions by region, device, codec, language, and network class. Optimize time to first useful audio and interruption response separately from total completion time.

Useful controls include:

- short audio frames and incremental processing;
- a bounded jitter buffer that adapts to observed network variation;
- streaming recognition or a speech-to-speech model;
- speculative response generation only when it is safe to discard;
- incremental speech synthesis;
- geographic media termination near the user;
- independent control and media channels so cancellation is not trapped behind audio backlog.

Never make queues unbounded to “avoid dropping data.” For a live session, a stale video frame or audio chunk may be worse than a dropped one. Preserve control events and committed records; shed obsolete media.

## When to use this architecture

Use a real-time multimodal agent when:

- the user is hands-busy or eyes-busy;
- conversational latency materially affects task completion;
- the agent must interpret speech together with a live camera, screen, or sensor stream;
- interruption, clarification, and rapid feedback are core interactions.

Prefer push-to-talk, recorded messages, or asynchronous workflows when connectivity is poor, auditability matters more than immediacy, the task needs careful review, or the agent could cause a high-impact action. “Real time” is a product requirement, not a default architecture.

## Transport and session protocol

WebRTC is usually the starting point for low-latency browser or mobile media because it includes real-time media transport, congestion handling, NAT traversal, and encrypted media. A WebSocket can be appropriate for control events, transcripts, and simpler server-mediated audio, but the application then owns more buffering, loss, and timing behavior.

A resumable session protocol should define:

- client and server event sequence numbers;
- acknowledgments and replay windows;
- a short-lived, session-bound resume token;
- deduplication of repeated events after reconnect;
- media timestamps independent of wall-clock time;
- heartbeat, idle, and maximum-session limits;
- explicit events such as `input.started`, `input.committed`, `response.started`, `playback.ack`, `response.cancelled`, and `tool.completed`.

A reconnect may restore context, but it must not replay already heard audio or repeat an outcome-unknown tool action.

## Turn detection and interruption

Silence is not the same as the end of a thought. Voice activity detection can identify speech, while semantic endpointing estimates whether the user has finished. Both fail with noise, accents, hesitations, radio traffic, and multiple speakers.

Provide a fallback such as push-to-talk or a visible “send” control. Treat partial transcripts as revisable. Never trigger an irreversible tool from an unstable transcript.

For barge-in:

1. The client reports the last played audio sample or segment.
2. A new user-speech event immediately cancels response generation and synthesis.
3. The server discards unsent output and marks only the acknowledged playback prefix as heard.
4. The conversation context retains that heard prefix, not the full planned answer.
5. Cancellable read-only tools receive cancellation.
6. A side-effecting tool that may already have executed moves to outcome reconciliation; cancellation must not be presented as rollback.

Use cancellation IDs and generation epochs so a late audio chunk or tool result from the old turn cannot enter the new turn.

## Tool and action safety

Speech is easy to misrecognize and may come from a bystander. Speaker recognition is a signal, not authorization.

For consequential actions:

- construct a canonical action payload from committed evidence;
- show or read back the exact resource, amount, destination, and effect;
- bind approval to the payload hash and source revision;
- require a deliberate confirmation through an appropriate channel;
- use an idempotency key and record an outcome ledger;
- never interpret “yes” from a tentative transcript as authorization;
- make read-only lookup and draft creation easier than submission.

A tool can finish after the user interrupts. The system must record its authoritative result and tell the user what happened rather than silently dropping the result.

## Multimodal alignment

Do not send every camera frame to the model. Sample according to task, motion, and scene change, then attach capture timestamps and provenance. Maintain explicit relationships between speech and visual evidence, such as “this utterance referred to frame 842 captured at 10:14:03.220.”

Failure to align time creates confident but stale answers: the user says “this valve” after the camera has moved, or the model reasons over a floor-plan revision that no longer matches the live view. Ask for a reframe or confirmation when alignment confidence is low.

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| WebRTC media | Low-latency media, congestion control, NAT traversal | More session and infrastructure complexity |
| WebSocket audio | Simpler application protocol | Application owns more timing, loss, and backpressure behavior |
| Aggressive endpointing | Faster responses | Cuts off hesitations and complex utterances |
| Conservative endpointing | Fewer false turns | Awkward pauses and higher latency |
| Speculative generation | Lower apparent latency | Wasted compute and greater cancellation complexity |
| Server-side raw media retention | Easier debugging and replay | Privacy, consent, residency, and breach exposure |
| Speech-to-speech model | Natural timing and fewer pipeline boundaries | Harder transcript audit and control over intermediate decisions |
| ASR + text model + TTS | Inspectable stages and interchangeable components | Accumulated latency and cross-stage error |

## Failure modes and defenses

| Failure mode | Design response |
|---|---|
| Packet loss, jitter, or reordering | Adaptive jitter buffer, sequence numbers, concealment, and telemetry |
| Echo causes the agent to hear itself | Acoustic echo cancellation plus server-side correlation and suppression |
| Partial transcript changes after a tool starts | Only committed text can authorize tool planning; actions require confirmation |
| User interrupts but old audio continues | Priority cancellation channel, generation epochs, and client playback flush |
| Tool finishes after cancellation | Outcome ledger and reconciliation message; never assume cancellation undid it |
| Reconnect repeats audio or action | Playback acknowledgments, event deduplication, and idempotency keys |
| Wrong speaker gives an instruction | Session authorization plus explicit confirmation; do not trust voice identity alone |
| Camera evidence is stale | Capture timestamps, freshness limits, and request a new frame |
| Media backlog grows | Bounded buffers and stale-frame shedding; protect control traffic |
| Cross-session media leakage | Per-session encryption, strict routing keys, worker cleanup, and isolation tests |
| Sensitive speech appears in logs | Redaction, metadata-first observability, short retention, and audited break-glass access |
| Prompt injection appears on a sign or speaker | Treat environmental content as untrusted evidence, not policy or authority |

## Construction example: hospital renovation field assistant

A superintendent walks through an active hospital renovation with a phone camera and headset. The assistant can identify the current drawing revision, retrieve an RFI, and draft a field issue while the user keeps both hands free.

The site is noisy and connectivity is intermittent. The design uses WebRTC for media, a nearby media edge, adaptive jitter buffering, and a control stream for acknowledgments and cancellation. Video is sampled on scene change and each frame is tagged with project, location, drawing revision, and capture time.

The superintendent asks, “Is this penetration approved?” The partial transcript first says “patient approved,” so no tool runs. After the utterance is committed, the agent retrieves the room-specific drawing and RFI history. It begins explaining, but the superintendent interrupts: “Stop—this is the west wall.” The old generation epoch is cancelled, unplayed audio is flushed, and the context records only what was heard. The new frame and location invalidate the earlier lookup.

The agent may draft an RFI with attached evidence. It cannot submit the RFI or change the schedule until the user reviews a canonical payload on screen and confirms it. If submission times out, the system reconciles against the authoritative project system instead of retrying blindly.

The same pattern applies to a healthcare operations assistant coordinating rooms or transport: obtain recording consent, minimize PHI, prevent ambient speech from becoming authorization, and keep clinical diagnosis outside the assistant’s scope.

## Interview prompt

**Design a real-time voice-and-camera assistant for construction field teams. It must tolerate noisy sites and intermittent networks, support user interruption, retrieve project evidence, and draft or submit RFIs without duplicate actions.**

## Worked answer

1. **Clarify requirements.** Ask about supported devices, languages, session concurrency, network conditions, latency objectives, retention, regions, tenant isolation, and which actions require approval.
2. **Define invariants.** No cross-tenant media, no action from tentative speech, no duplicate RFI, no replay of heard audio, and no claim that cancellation reversed an uncertain side effect.
3. **Separate planes.** Terminate WebRTC media near the user; send ordered control events to a regional session service; keep durable conversation and outcome records outside ephemeral media workers.
4. **Model session state.** Track tentative, committed, generated, delivered, and heard state with sequence numbers, timestamps, acknowledgments, and generation epochs.
5. **Build the media path.** Apply echo cancellation, adaptive jitter buffering, VAD plus semantic endpointing, incremental recognition/model inference, and streaming synthesis. Bound buffers and shed stale media.
6. **Handle barge-in.** Prioritize cancellation, flush client playback, discard unsent output, retain only the heard prefix, and isolate late results from the cancelled epoch.
7. **Ground multimodal answers.** Sample frames, attach location and revision metadata, align them with utterances by timestamp, and refuse or clarify when evidence is stale.
8. **Protect tools.** Use committed inputs, tenant-scoped authorization, canonical payload review, payload-bound approval, idempotency keys, and reconciliation for outcome-unknown writes.
9. **Recover safely.** Resume with short-lived tokens, deduplicate events, continue from playback acknowledgments, and never auto-repeat a consequential action.
10. **Observe and test.** Measure stage latency, interruption time, endpoint errors, packet loss, stale-frame use, cancellation races, duplicate attempts, and privacy violations. Test noise, overlapping speakers, network handoff, reconnect, and late tool completion.

**One-line conclusion:** A trustworthy real-time agent treats media as lossy and provisional, conversation state as acknowledged and replay-safe, and every consequential action as explicitly confirmed and idempotent.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [IETF RFC 8825 — Overview: Real-Time Protocols for Browser-Based Applications](https://www.rfc-editor.org/rfc/rfc8825)
- [IETF RFC 3550 — RTP: A Transport Protocol for Real-Time Applications](https://www.rfc-editor.org/rfc/rfc3550)
- [IETF RFC 7587 — RTP Payload Format for the Opus Speech and Audio Codec](https://www.rfc-editor.org/rfc/rfc7587)
- [W3C Media Capture and Streams](https://www.w3.org/TR/mediacapture-streams/)
- [IETF RFC 6455 — The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
