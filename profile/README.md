# WebRTT — Real Time Translation Protocol

An open protocol for low-latency voice translation between two people speaking different languages.

WebRTT brings conversation latency below 200ms by translating speculatively — before the speaker finishes their sentence — using a commit/revert model inspired by how human simultaneous interpreters work.

---

## The Problem

Current real-time translation pipelines are sequential:

```
voice → STT → text → MT → translated text → TTS → audio
```

Each step waits for the previous one to finish. The result is 600–1200ms of perceived delay — enough to make natural conversation feel broken.

## The Approach

WebRTT introduces **speculative translation**: the edge node begins translating and synthesizing audio before the sentence is complete, emitting `HYPOTHESIS` messages to the client. When the sentence ends, the node either `COMMIT`s the hypothesis (if it was correct) or `REVERT`s it (if not).

The result is a perceived latency under 200ms — the difference between a conversation that feels natural and one that feels like a phone call from 1995.

---

## Repositories

| Repo | Description |
|---|---|
| [`spec`](https://github.com/webrtt/spec) | The open protocol specification — RFC format, language-agnostic |
| [`webrtt`](https://github.com/webrtt/webrtt) | Reference implementation — Rust edge node, built for Cloudflare Workers via WASM |
| [`webrtt-sdk`](https://github.com/webrtt/webrtt-sdk) | TypeScript client SDK + Chrome extension for Google Meet |

---

## Protocol at a Glance

WebRTT runs over WebSocket. Messages are encoded with MessagePack.

```
┌─────────────────┐                         ┌─────────────────┐
│    Client A     │                         │    Client B     │
│   (speaker)     │                         │  (listener)     │
└────────┬────────┘                         └────────┬────────┘
         │                                           │
         │  SPEECH_START                             │
         │ ─────────────────────────────►            │
         │                                           │
         │  AUDIO_CHUNK(seq=1)                       │
         │ ─────────────────────────────►            │
         │                          ┌────────────────┴────────────────┐
         │                          │  STT: "hello, how..."  (partial) │
         │                          │  MT:  "olá, como..."  (speculate) │
         │                          │  TTS: begin synthesis             │
         │                          └────────────────┬────────────────┘
         │                                           │  HYPOTHESIS(conf=0.72)
         │                            ◄──────────────┤
         │                                           │  [client plays audio]
         │  AUDIO_CHUNK(seq=2)                       │
         │ ─────────────────────────────►            │
         │                          ┌────────────────┴────────────────┐
         │                          │  STT: "hello, how are you"       │
         │                          │  MT:  "olá, como vai você"       │
         │                          └────────────────┬────────────────┘
         │                                           │  HYPOTHESIS(conf=0.91)
         │                            ◄──────────────┤
         │  SPEECH_END                               │  [client updates audio]
         │ ─────────────────────────────►            │
         │                          ┌────────────────┴────────────────┐
         │                          │  STT: "hello, how are you?"      │
         │                          │  MT:  final — matches hypothesis  │
         │                          └────────────────┬────────────────┘
         │                                           │  COMMIT
         │                            ◄──────────────┤
         │                                           │  [audio confirmed]
```

Full specification: [`spec/RFC.md`](https://github.com/webrtt/spec/blob/main/RFC.md)

---

## Status

> **Draft — v0.1 in active development.**
> The protocol spec is open for discussion. Implementation is underway.
> We welcome feedback, issues, and RFC pull requests.

---

## Design Principles

**Open by default.** The protocol is a public spec. Any team can implement it in any language without depending on this organization.

**Latency is the product.** Every design decision is evaluated against its impact on end-to-end latency. A feature that adds 50ms is a feature that needs very strong justification.

**Rust for the runtime.** The reference edge node is written in Rust — no garbage collector, compiles to WASM, correct concurrent code by construction.

**Speculation over buffering.** We start translating before the sentence ends, like a human interpreter. We fix mistakes with REVERT, not by waiting.

---

## Contributing

The best place to start is [`spec/RFC.md`](https://github.com/webrtt/spec/blob/main/RFC.md).

If you want to propose a change to the protocol, open a pull request on the `spec` repo with your reasoning. Protocol changes require discussion before implementation.

For bugs and improvements in the reference implementation, open an issue on [`webrtt`](https://github.com/webrtt/webrtt).

---

## License

Protocol spec: [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/) — public domain, no restrictions.
Reference implementation: [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0)

---

*Built by [Surf No Digital](https://surfnodigital.com.br) — AI automation and infrastructure.*
