<div align="center">

# Awesome Voice Stack

[![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

*A curated guide to tools, architecture patterns, and tradeoffs for building production voice AI systems.*

</div>

## Contents

- [Architecture Patterns](#architecture-patterns)
- [Speech-to-Text (ASR)](#speech-to-text-asr)
- [Large Language Models](#large-language-models)
- [Text-to-Speech (TTS)](#text-to-speech-tts)
- [Voice Activity Detection (VAD)](#voice-activity-detection-vad)
- [Orchestration Frameworks](#orchestration-frameworks)
- [Real-time Transport: WebRTC](#real-time-transport-webrtc)
- [Real-time Transport: Telephony](#real-time-transport-telephony)
- [Real-time Transport: Edge and On-Device](#real-time-transport-edge-and-on-device)
- [Interruption and Turn-Taking](#interruption-and-turn-taking)
- [Latency Optimization](#latency-optimization)
- [Evaluation and Benchmarking](#evaluation-and-benchmarking)
- [Observability and Debugging](#observability-and-debugging)
- [Cost Analysis and Optimization](#cost-analysis-and-optimization)
- [Multilingual and Localization](#multilingual-and-localization)
- [Compliance, Security, and Privacy](#compliance-security-and-privacy)

## Architecture Patterns

Every production voice AI system starts with a fundamental design choice: how tightly to couple the speech and language understanding stages.

### Cascading Pipeline

The most common production pattern. Each stage is an independent, swappable component.

```
Mic → [VAD] → [ASR] → text → [LLM] → text → [TTS] → Speaker
```

| Advantage | Disadvantage |
|---|---|
| Swap any component independently | Cumulative latency across stages |
| Mature tooling at every layer | Loss of paralinguistic cues (tone, emotion) between stages |
| Easier to debug and observe | Each boundary is a serialization point |
| Fine-grained cost control | Text bottleneck loses audio nuance |

### End-to-End Speech-to-Speech

A single model ingests audio and produces audio directly, preserving tone, emotion, and prosody.

- [GPT-4o Realtime API](https://platform.openai.com/docs/guides/realtime) - OpenAI's native audio-in/audio-out model with built-in voice.
- [Gemini Live](https://deepmind.google/technologies/gemini/live/) - Google's multimodal model with native audio streaming.
- [Moshi](https://github.com/kyutai-labs/moshi) - Open-weight speech-to-speech model from Kyutai with full-duplex conversation.
- [Ultravox](https://github.com/fixie-ai/ultravox) - Open-weight audio-native LLM that understands speech directly without a separate ASR step.
- [Qwen3-Omni](https://github.com/QwenLM/Qwen3-Omni) - End-to-end omni-modal model from Alibaba with native speech understanding and generation.
- [MiniCPM-o](https://huggingface.co/openbmb/MiniCPM-o-2_6) - Compact end-to-end multimodal model with speech capabilities.

| Advantage | Disadvantage |
|---|---|
| Potentially lower end-to-end latency | Limited model choices today |
| Preserves tone, emotion, and prosody | Hard to inspect or debug intermediate steps |
| Simpler deployment (fewer services) | Less control over voice selection and style |
| Can handle non-verbal cues | Higher compute cost per request |

### Hybrid Approaches

Use an audio-native LLM for understanding (skip ASR) but a separate TTS for output. Balances latency and control.

- [Ultravox + Cartesia](https://docs.ultravox.ai/) - Audio-native understanding with high-quality streaming TTS output.
- [LFM2-audio](https://www.liquid.ai/) - Liquid AI's foundation model with native audio understanding and separate speech output.

### Choosing a Pattern

| Factor | Cascading | End-to-End | Hybrid |
|---|---|---|---|
| Production readiness | High | Medium | Medium |
| Latency floor | ~800ms | ~300ms | ~500ms |
| Component flexibility | Full | None | Partial |
| Voice quality control | Full | Limited | Full |
| Debugging ease | High | Low | Medium |
| Cost transparency | High | Opaque | Medium |

## Speech-to-Text (ASR)

Converts user speech to text. In a cascading pipeline, ASR is the first bottleneck — its speed and accuracy set the ceiling for everything downstream.

### Key Tradeoffs

- **Streaming vs. batch** — Streaming ASR emits partial transcripts as the user speaks (~200ms chunks), enabling faster LLM processing. Batch ASR waits for the full utterance and is more accurate but adds latency. Use streaming for real-time conversation, batch for post-call analytics.
- **Accuracy vs. latency** — Larger models (Whisper Large, Parakeet 1.1B) have lower word error rates but higher compute cost. Smaller models (Whisper Tiny, Moonshine) run on CPU but trade accuracy.
- **Cloud vs. self-hosted** — Cloud APIs are simpler to operate but introduce network round-trip latency and per-minute costs. Self-hosted models eliminate API costs at scale but require GPU infrastructure.

### Cloud Providers

- [Deepgram](https://deepgram.com/) - Streaming-first ASR with Nova-3 models. Sub-300ms latency, strong accuracy, and WebSocket API. From $0.0058/min.
- [Gradium](https://gradium.ai/) - Streaming speech-to-text API with semantic turn detection.
- [AssemblyAI](https://www.assemblyai.com/) - High-accuracy ASR with Universal-2 model, real-time streaming, and built-in speaker diarization.
- [Google Cloud Speech-to-Text](https://cloud.google.com/speech-to-text) - Chirp 2 model with 100+ language support and streaming recognition. $0.016/min.
- [Azure Speech Service](https://azure.microsoft.com/en-us/products/ai-services/speech-to-text) - Fast Speech recognition with custom model training and real-time streaming. $1/hour.
- [OpenAI Whisper API](https://platform.openai.com/docs/guides/speech-to-text) - Hosted Whisper with simple API. Batch-only (no streaming). $0.006/min.
- [AWS Transcribe](https://aws.amazon.com/transcribe/) - Streaming and batch ASR with custom vocabulary support. $0.024/min.
- [Cartesia Ink-Whisper](https://cartesia.ai/) - Streaming STT with 66ms median time-to-complete-transcript. Integrated with Cartesia's voice platform.

### Open-Source Models

- [Whisper](https://github.com/openai/whisper) - OpenAI's foundational ASR model. 99 languages. Multiple sizes from 39M to 1.5B parameters. The accuracy benchmark others compete against.
- [Faster-Whisper](https://github.com/SYSTRAN/faster-whisper) - CTranslate2-based Whisper reimplementation. Up to 4x faster than original with comparable accuracy.
- [Whisper.cpp](https://github.com/ggerganov/whisper.cpp) - C/C++ port of Whisper optimized for CPU and edge inference. Runs on Raspberry Pi and phones.
- [Distil-Whisper](https://github.com/huggingface/distil-whisper) - Distilled Whisper models from Hugging Face. 6x faster, 49% smaller, within 1% WER of the original.
- [Moonshine](https://github.com/moonshine-ai/moonshine) - Edge-optimized ASR from Useful Sensors. 245M params beats Whisper Large V3 (1.5B) on accuracy. 5x faster. MIT licensed.
- [Parakeet](https://github.com/NVIDIA/NeMo) - NVIDIA's state-of-the-art ASR via NeMo. 600M params, #1 on HuggingFace ASR leaderboard with 6.05% WER. FastConformer architecture.
- [WhisperX](https://github.com/m-bain/whisperX) - Whisper with word-level timestamps, speaker diarization, and VAD-based chunking. Best for post-processing workflows.
- [Wav2Vec 2.0](https://huggingface.co/facebook/wav2vec2-large-960h) - Meta's self-supervised speech model. Good baseline for fine-tuning on domain-specific data.

### When to Use What

| Scenario | Recommended Approach |
|---|---|
| Real-time voice agent | Streaming cloud ASR (Deepgram, AssemblyAI) or self-hosted Faster-Whisper |
| Post-call transcription | Batch Whisper Large or Parakeet for maximum accuracy |
| On-device / offline | Whisper.cpp, Moonshine, or Sherpa-ONNX |
| Multilingual product | Whisper (99 languages) or Google Chirp 2 |
| Cost-sensitive at scale | Self-hosted Faster-Whisper or Distil-Whisper on GPU |

## Large Language Models

The reasoning layer. In voice AI, LLM selection is dominated by **time-to-first-token (TTFT)** — users perceive anything over ~1 second of silence as lag.

### Key Tradeoffs

- **TTFT vs. intelligence** — Smaller, faster models (GPT-4o-mini, Gemini Flash, Claude Haiku) respond in 200-400ms but may struggle with complex reasoning. Larger models are smarter but slower.
- **Streaming output** — Streaming token-by-token into TTS is essential. A model that returns the full response at once adds unacceptable latency.
- **Function calling** — Voice agents that book appointments, query databases, or transfer calls need reliable structured output. Not all models handle tool use equally well.
- **Context window management** — Voice conversations accumulate context quickly. Summarization or sliding windows prevent cost blowup on long calls.

### Cloud LLMs for Voice

Optimized for low-latency, streaming use cases:

- [GPT-4o-mini](https://platform.openai.com/docs/models) - Fast, cheap, good function calling. The workhorse for most voice agents.
- [GPT-4o](https://platform.openai.com/docs/models) - Higher intelligence when needed. Also supports native audio input/output via Realtime API.
- [Claude 3.5 Haiku](https://www.anthropic.com/claude) - Anthropic's fastest model. Strong instruction following, low TTFT.
- [Claude 3.5 Sonnet](https://www.anthropic.com/claude) - Balance of speed and capability for complex voice workflows.
- [Gemini 2.0 Flash](https://deepmind.google/technologies/gemini/) - Google's speed-optimized model. Native audio understanding. Very competitive TTFT.
- [Groq](https://groq.com/) - LPU-accelerated inference for open models. Sub-100ms TTFT for Llama and Mistral.
- [Cerebras](https://cerebras.ai/) - Wafer-scale inference for open models with extremely fast token generation.

### Self-Hosted LLMs

For cost control at scale, data privacy, or offline operation:

- [Llama 3](https://github.com/meta-llama/llama3) - Meta's open-weight models. 8B and 70B variants. Strong community and tooling.
- [Mistral / Mixtral](https://mistral.ai/) - Efficient open models with strong multilingual performance.
- [Qwen 2.5](https://github.com/QwenLM/Qwen2.5) - Alibaba's open models. Competitive with Llama 3. Strong multilingual and tool use.
- [vLLM](https://github.com/vllm-project/vllm) - High-throughput serving engine with PagedAttention. The standard for self-hosted LLM inference.
- [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM) - NVIDIA's optimized inference library. Best throughput on NVIDIA GPUs.
- [llama.cpp](https://github.com/ggerganov/llama.cpp) - CPU and mixed-precision inference. Runs on consumer hardware and edge devices.

### Voice-Specific LLM Considerations

- **System prompts** — Voice agents need prompts that encourage concise, conversational responses. Long-winded answers feel unnatural when spoken.
- **Response length control** — Instruct the model to keep responses under 2-3 sentences for most turns. Users can always ask for more detail.
- **Filler tokens** — Some systems prompt the LLM to emit a filler word ("Let me check...") before tool calls to fill silence.
- **Turn-aware context** — Include conversation history but trim aggressively. Voice context is more ephemeral than chat context.

## Text-to-Speech (TTS)

Converts LLM output to spoken audio. In production, the critical metric is **time-to-first-audio (TTFA)** — how quickly the user hears the first syllable.

### Key Tradeoffs

- **Naturalness vs. latency** — More natural-sounding models tend to be larger and slower. For real-time voice agents, a slightly less natural voice at 50ms TTFA often beats a perfect voice at 500ms.
- **Streaming support** — TTS must accept text token-by-token from the LLM stream and produce audio chunks incrementally. Batch TTS that waits for the full sentence is unusable for real-time conversation.
- **Voice cloning vs. preset voices** — Cloned voices require extra infrastructure and raise ethical/legal considerations. Preset voices are simpler to deploy and more predictable.

### Cloud Providers

- [Cartesia Sonic](https://cartesia.ai/) - Purpose-built for real-time voice AI. Sonic Turbo: 40ms TTFA. 42 languages. Emotion and laughter tags. The latency benchmark.
- [Gradium](https://gradium.ai/) - Streaming text-to-speech API for voice applications.
- [ElevenLabs](https://elevenlabs.io/) - Highest naturalness ratings. Excellent voice cloning. Turbo v2.5 model for low-latency streaming. 32 languages.
- [OpenAI TTS](https://platform.openai.com/docs/guides/text-to-speech) - Simple API with 6 preset voices. Good quality, easy integration. No streaming in standard API.
- [Play.ht](https://play.ht/) - Real-time streaming TTS with voice cloning. Play3.0 model with emotion control. Good multilingual support.
- [Deepgram Aura](https://deepgram.com/aura) - Streaming TTS designed for conversational AI. Low latency, pay-per-character pricing.
- [Rime](https://rime.ai/) - Fast TTS optimized for voice agents. Sub-100ms latency with Mist v2. Good for telephony-grade output.
- [Azure Neural TTS](https://azure.microsoft.com/en-us/products/ai-services/text-to-speech) - Wide language coverage (140+ languages). SSML control. Custom Neural Voice for enterprise voice cloning.

### Open-Source Models

- [Kokoro](https://github.com/hexgrad/kokoro) - 82M parameter model that punches above its weight. Based on StyleTTS 2 architecture. Fast inference, Apache 2.0 licensed. Supports English, French, Korean, Japanese, Mandarin.
- [Piper](https://github.com/rhasspy/piper) - Optimized for local and edge inference. Runs on Raspberry Pi. Multiple quality tiers. Great for on-device deployment.
- [Orpheus](https://github.com/canopyai/Orpheus-TTS) - LLM-based TTS built on Llama architecture. Natural prosody with emotion tags (laughter, hesitation). 3B parameter model.
- [StyleTTS 2](https://github.com/yl4579/StyleTTS2) - Diffusion-based style transfer. Near-human quality on single-speaker benchmarks. Higher compute requirement.
- [F5-TTS](https://github.com/SWivid/F5-TTS) - Flow-matching-based TTS with zero-shot voice cloning. Good balance of quality and inference speed.
- [Dia](https://github.com/nari-labs/dia) - Dialogue-oriented TTS from Nari Labs. Generates multi-speaker audio with non-verbal cues (laughter, pauses). 1.6B parameters.
- [Bark](https://github.com/suno-ai/bark) - Text-to-audio model by Suno. Generates speech with non-verbal sounds. Multilingual but slower inference.
- [Coqui TTS](https://github.com/coqui-ai/TTS) - Comprehensive TTS toolkit with multiple model architectures and voice cloning. Community-maintained since Coqui's closure.

### Streaming TTS Considerations

For production voice AI, TTS must operate in a streaming pipeline:

1. Receive text tokens from LLM stream as they arrive.
2. Buffer tokens until a speakable chunk is formed (phrase or sentence boundary).
3. Synthesize that chunk and begin audio playback immediately.
4. Continue synthesizing subsequent chunks while earlier ones play.

The chunk boundary strategy matters: too small (word-level) produces choppy prosody; too large (full sentence) adds latency. Most production systems chunk at sentence or clause boundaries.

## Voice Activity Detection (VAD)

Determines when a user starts and stops speaking. In production, VAD quality directly impacts turn-taking behavior — a bad VAD causes the agent to cut off users or wait too long to respond.

### Why It Matters

- **Endpointing** — Detecting when the user has finished their turn. Too aggressive = cuts off mid-sentence. Too conservative = long pauses before response.
- **Barge-in** — Detecting when the user starts speaking over the agent. Requires fast onset detection.
- **Noise rejection** — Filtering background noise, music, and crosstalk so they do not trigger false positives in ASR.
- **Cost savings** — Only sending audio to ASR when speech is detected reduces API costs significantly.

### Tools

- [Silero VAD](https://github.com/snakers4/silero-vad) - Pre-trained ONNX model. Fast, lightweight, runs on CPU. The de facto standard for open-source voice pipelines.
- [WebRTC VAD](https://webrtc.org/) - Google's VAD from the WebRTC project. Very lightweight, C-based. Less accurate than Silero but minimal compute overhead.
- [Cobra](https://picovoice.ai/platform/cobra/) - Picovoice's commercial VAD. Cross-platform, on-device, optimized for low false-positive rates.
- [Moonshine VAD](https://github.com/moonshine-ai/moonshine) - Bundled with Moonshine ASR. On-device VAD with integrated ASR pipeline.

### Tradeoffs

| Setting | More Aggressive | More Conservative |
|---|---|---|
| Endpointing silence threshold | Faster responses, more interruptions | Fewer interruptions, slower turn transitions |
| Onset sensitivity | Catches soft speech, more false triggers | Misses mumbles, fewer false triggers |
| Noise gate level | Rejects more noise, may clip quiet speech | Passes more noise to ASR, fewer missed words |

Most production systems use adaptive endpointing: shorter silence thresholds for yes/no questions, longer for open-ended prompts.

## Orchestration Frameworks

Orchestrators glue the pipeline together. They manage audio streaming, model routing, interruption handling, state machines, and conversation flow.

### Key Tradeoffs

- **Open-source vs. managed platform** — Open-source frameworks (Pipecat, LiveKit Agents) give full control and no per-minute fees but require you to build and operate the infrastructure. Managed platforms (VAPI, Retell, Bland) handle scaling, telephony, and observability but charge per-minute and limit customization.
- **Build vs. buy** — If your product's differentiator is the voice experience, build on open-source. If voice is a feature of a larger product, a managed platform gets you to market faster.

### Open-Source Frameworks

- [Pipecat](https://github.com/pipecat-ai/pipecat) - Python framework with a pipeline-of-frames architecture. 15+ provider integrations swappable with one line of code. WebRTC and WebSocket transports. 500-800ms typical latency. Active community (10k+ stars).
- [LiveKit Agents](https://github.com/livekit/agents) - Python and Node.js framework built on LiveKit's WebRTC infrastructure. Tight integration with LiveKit Cloud for managed deployment. Plugin system for ASR, LLM, and TTS providers.
- [Vocode](https://github.com/vocodedev/vocode-core) - Python library for voice agents with phone call support (Twilio integration), Zoom, and web. Abstracts transcriber, agent, and synthesizer components.
- [Bolna](https://github.com/bolna-ai/bolna) - Open-source voice AI agent framework with telephony support. Integrates ASR, LLM, and TTS with conversation management.

### Managed Platforms

- [VAPI](https://vapi.ai/) - Voice AI platform with phone number provisioning, function calling, and analytics. Pay-per-minute pricing. Good for rapid prototyping.
- [Retell AI](https://retellai.com/) - Conversational voice AI platform with low-latency pipeline and built-in telephony. Custom LLM support.
- [Bland AI](https://bland.ai/) - Enterprise voice AI for phone calls. Hyper-realistic voices, high concurrency. Focused on outbound calling use cases.
- [Synthflow](https://synthflow.ai/) - No-code voice agent builder. Templates for common use cases (booking, support). Good for non-technical teams.

### Decision Framework

| Factor | Open-Source | Managed Platform |
|---|---|---|
| Time to first demo | Days | Hours |
| Per-minute cost | Infrastructure only | $0.05-0.15/min + provider costs |
| Customization depth | Unlimited | Platform-constrained |
| Telephony integration | DIY (Twilio, Telnyx) | Built-in |
| Scaling responsibility | You | Platform |
| Data control | Full | Platform-dependent |

## Real-time Transport: WebRTC

WebRTC is the standard for browser-based and app-based voice AI. It provides sub-200ms transport latency, NAT traversal, built-in echo cancellation, and noise suppression.

### Key Tradeoffs

- **Managed vs. self-hosted** — Managed WebRTC (Daily.co, LiveKit Cloud) eliminates TURN/STUN server management and global distribution. Self-hosted (LiveKit OSS, Mediasoup) avoids per-minute fees at scale but requires infrastructure expertise.
- **SFU vs. peer-to-peer** — Voice AI always needs a Selective Forwarding Unit (SFU) since the AI agent is a server-side participant. Pure P2P is not applicable.

### Managed Providers

- [LiveKit Cloud](https://livekit.io/) - Managed WebRTC with built-in voice agent support. Deploys agents as room participants. Integrated with LiveKit Agents framework. $0.0018/min.
- [Daily.co](https://daily.co/) - WebRTC platform with Pipecat integration. Simple REST API for room management. $0.004/min.
- [100ms](https://100ms.live/) - WebRTC infrastructure with low-latency audio. SDKs for web, iOS, Android. $0.004/min.
- [Agora](https://agora.io/) - Global real-time engagement platform. Extensive SDK support. $0.99/1000 min.

### Self-Hosted

- [LiveKit (OSS)](https://github.com/livekit/livekit) - Open-source WebRTC SFU written in Go. Production-grade, horizontally scalable. The most popular choice for self-hosted voice AI transport.
- [Mediasoup](https://github.com/versatica/mediasoup) - C++ WebRTC SFU with Node.js API. Flexible and performant. Requires more low-level configuration.
- [Pion](https://github.com/pion/webrtc) - Pure Go WebRTC implementation. Building block for custom transport solutions rather than a turnkey SFU.

### When to Self-Host

Self-hosting makes economic sense when:

- Sustained concurrency exceeds ~50 simultaneous sessions.
- Data sovereignty requirements prevent using US-based cloud providers.
- You need custom audio processing in the media path (custom VAD, audio watermarking).

## Real-time Transport: Telephony

Telephony connects voice AI to the existing phone network (PSTN). This is essential for call centers, appointment booking, and outbound calling use cases.

### Key Tradeoffs

- **Codec quality** — PSTN uses G.711 (8kHz, narrowband). WebRTC uses Opus (48kHz, wideband). Voice AI models trained on wideband audio may perform worse on narrowband telephony input. Some providers offer Opus passthrough on SIP trunks.
- **Latency** — PSTN adds 50-150ms of transport latency on top of your pipeline. Budget for this.
- **Regulatory** — Outbound calling has strict compliance requirements (STIR/SHAKEN, TCPA, do-not-call lists). AI disclosure laws are evolving.

### SIP Trunking Providers

- [Twilio](https://www.twilio.com/voice) - The market default. Elastic SIP Trunking, phone number management, global reach. $0.008/min. Extensive documentation and SDK support.
- [Telnyx](https://telnyx.com/) - Developer-focused telephony with Mission Control portal. Lower pricing than Twilio. Good API design. $0.005/min.
- [Vonage](https://www.vonage.com/) - CPaaS platform with SIP trunking and voice APIs. Strong international coverage.
- [SignalWire](https://signalwire.com/) - Built by FreeSWITCH creators. Programmable voice with AI integrations. Competitive pricing for high-volume use cases.
- [Plivo](https://www.plivo.com/) - Voice and SMS API platform with SIP trunking. Cost-effective for outbound calling.

### Telephony Integration Patterns

1. **SIP INVITE to voice pipeline** — Incoming calls hit a SIP trunk, media is forwarded to your voice AI pipeline via RTP or WebSocket.
2. **WebSocket audio streaming** — Twilio Media Streams and Telnyx RT forward call audio over WebSocket, enabling easy integration with cloud-based pipelines.
3. **SIP REFER for transfers** — Hand off calls to human agents or other systems using SIP REFER.

### Codec Considerations

| Codec | Sample Rate | Bitrate | Quality | Use Case |
|---|---|---|---|---|
| G.711 (PCMU/PCMA) | 8kHz | 64 kbps | Narrowband | PSTN standard, universal compatibility |
| G.722 | 16kHz | 64 kbps | Wideband | Better quality over PSTN, not universal |
| Opus | 8-48kHz | 6-510 kbps | Adaptive | WebRTC standard, best quality |

If your ASR model was trained on 16kHz audio (most are), G.711's 8kHz input will degrade accuracy. Consider upsampling or using models fine-tuned for narrowband audio.

## Real-time Transport: Edge and On-Device

Running voice AI locally eliminates network latency and cloud costs, and enables offline operation. The tradeoff is reduced model capability and increased device requirements.

### Key Tradeoffs

- **Privacy vs. capability** — On-device processing means audio never leaves the device. But edge models are smaller and less capable than cloud counterparts.
- **Bandwidth vs. compute** — Sending audio to the cloud uses bandwidth; processing locally uses device CPU/GPU. Choose based on your deployment constraints.
- **Model size vs. quality** — Sub-100MB models run on phones and Raspberry Pis but sacrifice accuracy and naturalness. Larger models need dedicated hardware.

### On-Device ASR

- [Whisper.cpp](https://github.com/ggerganov/whisper.cpp) - C/C++ Whisper inference. Runs on CPU, Apple Silicon (Core ML), and Android. The go-to for on-device transcription.
- [Sherpa-ONNX](https://github.com/k2-fsa/sherpa-onnx) - ONNX-based speech toolkit. Supports streaming and non-streaming ASR on CPU. Cross-platform (iOS, Android, Raspberry Pi, embedded Linux).
- [Moonshine](https://github.com/moonshine-ai/moonshine) - Models as small as 26MB. Runs on Raspberry Pi and IoT devices. Includes bundled VAD and speaker identification.
- [Vosk](https://github.com/alphacep/vosk-api) - Offline speech recognition for 20+ languages. Small models (50MB). C, Python, Java, and JavaScript APIs.

### On-Device TTS

- [Piper](https://github.com/rhasspy/piper) - ONNX-based TTS optimized for Raspberry Pi. Multiple voice quality tiers. Real-time on single-core CPU.
- [Kokoro](https://github.com/hexgrad/kokoro) - 82M parameter model with ONNX export. Fast enough for real-time on mid-range mobile devices.
- [eSpeak NG](https://github.com/espeak-ng/espeak-ng) - Formant-based speech synthesizer. Tiny footprint, 100+ languages. Robotic but usable for accessibility and embedded systems.
- [Sherpa-ONNX TTS](https://github.com/k2-fsa/sherpa-onnx) - Includes VITS-based TTS models for on-device synthesis. Same cross-platform support as its ASR counterpart.

### On-Device LLM

- [llama.cpp](https://github.com/ggerganov/llama.cpp) - Run quantized LLMs on CPU or Apple Silicon. 4-bit quantized Llama 3 8B runs on 8GB RAM devices.
- [MLC LLM](https://github.com/mlc-ai/mlc-llm) - Universal LLM deployment on phones, browsers, and edge devices. Supports iOS, Android, and WebGPU.
- [Ollama](https://github.com/ollama/ollama) - Simple local LLM runner. Good developer experience but higher overhead than llama.cpp.

### Deployment Targets

| Target | ASR | LLM | TTS | Realistic? |
|---|---|---|---|---|
| Raspberry Pi 5 | Moonshine (26MB) | Phi-3 mini (4-bit) | Piper (low quality) | Functional but slow |
| iPhone / Android (flagship) | Whisper.cpp (small) | Llama 3 8B (4-bit) | Kokoro | Usable for simple agents |
| Apple Silicon Mac / Desktop GPU | Faster-Whisper (medium) | Llama 3 70B (4-bit) | Orpheus | Full-quality local stack |
| NVIDIA Jetson | Moonshine or Whisper.cpp | Llama 3 8B | Piper or Kokoro | Good for kiosk/robotics |

## Interruption and Turn-Taking

The hardest problem in production voice AI. Humans naturally overlap, interject, and use backchannels ("mm-hmm", "right"). Getting this wrong makes a voice agent feel robotic or frustrating.

### Core Challenges

- **Endpointing** — Deciding the user has finished speaking. Too fast triggers on pauses mid-sentence. Too slow creates awkward silence.
- **Barge-in** — The user starts speaking while the agent is talking. The agent must detect this, stop its audio, and listen.
- **Backchanneling** — The user says "uh-huh" or "okay" during agent speech. This should not be treated as an interruption.
- **False triggers** — Background noise, coughs, and crosstalk can look like speech onset.

### Strategies

**Endpointing approaches:**
- Fixed silence threshold (e.g., 700ms of silence = end of turn). Simple but fragile.
- Adaptive threshold based on conversation context. Shorter for yes/no questions, longer for open-ended prompts.
- LLM-based endpointing: feed the partial transcript to a small model that predicts whether the user is done. More accurate but adds latency.
- Prosodic cues: falling intonation and slowing tempo suggest turn completion. Requires audio-level analysis.

**Barge-in handling:**
- Immediately stop TTS playback and discard queued audio when user speech onset is detected.
- Feed interrupted context back to the LLM so it knows the user cut in.
- Debounce barge-in detection (require 200-300ms of sustained speech) to filter coughs and noise.

**Backchannel detection:**
- Classify short utterances ("mm-hmm", "right", "okay") as backchannels rather than interruptions.
- Use a small classifier on the VAD output to distinguish short affirmations from real interruptions.

### Production Tips

- Log all interruption events with timestamps and audio. This data is invaluable for tuning thresholds.
- A/B test endpointing thresholds. The optimal value varies by use case (customer support vs. sales vs. booking).
- Consider the user's emotional state. Frustrated users speak faster and interrupt more — tighten barge-in sensitivity.
- Test with real background noise (TV, car, cafe). Lab conditions are misleading.

## Latency Optimization

Users perceive voice AI latency as the time between finishing their sentence and hearing the agent's first word. The target for natural conversation is **under 500ms** end-to-end, though under 1 second is acceptable for most use cases.

### The Latency Budget

```
User stops speaking
  └─ Endpointing delay         ~300-700ms  (VAD silence threshold)
  └─ ASR finalization           ~100-300ms  (final transcript after silence)
  └─ LLM time-to-first-token   ~200-500ms  (depends on model and provider)
  └─ TTS time-to-first-audio   ~50-200ms   (text chunk → first audio frame)
  └─ Network transport          ~20-100ms   (WebRTC/WebSocket)
────────────────────────────────────────────
  Total perceived latency       ~700-1800ms
```

### Optimization Techniques

**Pipeline-level:**
- Stream at every stage. ASR partial transcripts feed the LLM before the user finishes speaking. LLM tokens feed TTS as they arrive.
- Speculative execution: start LLM inference on partial ASR transcript and discard/restart if the final transcript differs significantly.
- Pre-fetch: for predictable flows (menus, confirmations), pre-generate TTS audio for likely responses.
- Reduce endpointing delay: use an adaptive or LLM-assisted endpointer instead of a fixed long silence threshold.

**ASR optimization:**
- Use streaming ASR with low chunk intervals (~200ms).
- Deploy ASR in the same region as your LLM to minimize inter-service latency.
- Consider Distil-Whisper or Moonshine for faster inference with acceptable accuracy loss.

**LLM optimization:**
- Choose models with low TTFT (GPT-4o-mini, Gemini Flash, Groq-hosted Llama).
- Keep prompts concise. Every extra token in the system prompt adds to TTFT.
- Use response prefill or guided generation to reduce first-token latency.

**TTS optimization:**
- Choose TTS with low TTFA (Cartesia Sonic Turbo: 40ms, Rime Mist: sub-100ms).
- Buffer 1-2 sentence-boundary chunks before synthesizing. Too-small chunks produce unnatural prosody.
- Pre-warm TTS connections to avoid cold-start latency.

**Infrastructure:**
- Co-locate services. ASR, LLM, and TTS in the same cloud region or even on the same GPU.
- Use HTTP/2 or persistent WebSocket connections to avoid connection setup overhead.
- Edge deployment for the SFU/media server to minimize user-to-server round trip.

### Benchmarking

Measure end-to-end latency from the user's last audio frame to the first audio frame of the agent's response. Break it down by component:

| Metric | Target | How to Measure |
|---|---|---|
| Endpointing delay | < 500ms | Timestamp from last speech frame to end-of-turn signal |
| ASR finalization | < 200ms | Timestamp from end-of-turn to final transcript |
| LLM TTFT | < 300ms | Timestamp from prompt sent to first token received |
| TTS TTFA | < 100ms | Timestamp from first text chunk to first audio frame |
| Transport (one-way) | < 50ms | WebRTC stats API or synthetic RTP probes |
| **Total** | **< 1000ms** | End-to-end measurement |

## Evaluation and Benchmarking

Voice AI systems are harder to evaluate than text-based systems. Quality is multi-dimensional: transcription accuracy, response relevance, voice naturalness, latency, and overall conversational feel.

### What to Measure

**Component-level metrics:**
- **Word Error Rate (WER)** — ASR accuracy. Measure on your domain-specific audio, not generic benchmarks. A 5% WER on LibriSpeech may be 15% on your actual call audio.
- **Mean Opinion Score (MOS)** — TTS quality rated by humans on a 1-5 scale. Expensive but irreplaceable for voice quality assessment.
- **Time-to-first-token (TTFT)** — LLM responsiveness.
- **Time-to-first-audio (TTFA)** — TTS streaming latency.
- **End-to-end latency** — Overall response time perceived by the user. The metric that matters most.

**System-level metrics:**
- **Task completion rate** — Did the voice agent accomplish the user's goal (booking, transfer, information retrieval)?
- **Turn count** — How many back-and-forth turns to complete a task. Lower is generally better.
- **Interruption rate** — How often does the user barge in? High rates suggest the agent is too verbose or too slow.
- **Abandonment rate** — How often does the user hang up or disengage before task completion?
- **Escalation rate** — How often does the agent need to transfer to a human?

### Tools

- [Braintrust](https://braintrust.dev/) - Evaluation and observability platform with support for voice AI pipelines. LLM-as-judge and human evaluation workflows.
- [Humanloop](https://humanloop.com/) - Prompt management and evaluation. A/B testing for LLM responses.
- [Ragas](https://github.com/explodinggradients/ragas) - Open-source evaluation framework for RAG systems. Adaptable for voice AI knowledge retrieval accuracy.
- [LangSmith](https://smith.langchain.com/) - LangChain's tracing and evaluation platform. Useful for debugging LLM behavior in voice pipelines.
- Custom metrics — Most teams build internal dashboards tracking latency percentiles, task completion, and user satisfaction (CSAT/NPS post-call surveys).

### Evaluation Approaches

- **LLM-as-judge** — Use a capable LLM to rate transcript quality, response appropriateness, and conversation flow. Fast and scalable but biased toward fluency over correctness.
- **Human evaluation** — Gold standard for voice quality (MOS) and conversation quality. Expensive. Use for periodic audits, not continuous testing.
- **Regression testing** — Maintain a set of recorded conversations with expected outcomes. Run your pipeline against them on every deployment. Catch regressions before production.
- **A/B testing** — Route a percentage of live traffic to a variant (different ASR, LLM, TTS, or prompt). Compare task completion rate and user satisfaction. Essential for data-driven optimization.

## Observability and Debugging

Voice pipelines are uniquely difficult to debug. Audio is opaque, latency issues are intermittent, and failures cascade silently across stages.

### Challenges

- **Audio is hard to log and search.** You cannot grep an audio file. Transcripts are a lossy proxy. Debugging ASR errors requires listening to the actual audio.
- **Distributed tracing across modalities.** A single turn crosses audio capture, VAD, ASR, LLM, TTS, and audio playback — potentially on different services and machines.
- **Latency attribution.** "The agent was slow" could be ASR, LLM, TTS, network, or endpointing. You need per-component timestamps.
- **Intermittent failures.** Audio quality varies by device, network, and environment. Issues that reproduce in 2% of calls are hard to catch in testing.

### What to Log

For every conversational turn:

- Raw audio (input and output) with timestamps.
- VAD events (speech onset, speech offset, endpointing trigger).
- ASR transcript (partial and final) with confidence scores and timestamps.
- LLM prompt and response with token-level timestamps.
- TTS input text and output audio duration.
- Interruption events with cause classification.
- Per-component latency breakdown.

### Tools

- [LangSmith](https://smith.langchain.com/) - Trace LLM calls with input/output pairs, latency, and cost. Good for debugging prompt and model issues.
- [Langfuse](https://langfuse.com/) - Open-source LLM observability. Tracing, prompt management, and evaluation. Self-hostable.
- [OpenTelemetry](https://opentelemetry.io/) - Vendor-neutral distributed tracing. Use custom spans for each pipeline stage. Export to Jaeger, Datadog, or Grafana.
- [Datadog](https://www.datadoghq.com/) - Full-stack observability with APM and custom metrics. Good for infrastructure-level monitoring of GPU utilization and queue depths.
- [Grafana + Prometheus](https://grafana.com/) - Open-source metrics and dashboards. Build custom voice pipeline dashboards tracking latency percentiles and throughput.

### Debugging Workflow

1. **Reproduce from logs.** Store enough context (audio, transcripts, LLM I/O) to replay any conversation turn offline.
2. **Component-level bisection.** Replace each component with a known-good fixture (canned audio, fixed transcript, deterministic LLM response) to isolate which stage is failing.
3. **Latency flame graphs.** Visualize per-component latency across many requests to identify outliers and bottlenecks.
4. **Audio quality triage.** When ASR accuracy drops, listen to the source audio. Common culprits: echo, background noise, codec artifacts, and microphone clipping.

## Cost Analysis and Optimization

Voice AI costs are dominated by three components: ASR, LLM inference, and TTS. Transport and infrastructure costs are secondary but grow with concurrency.

### Cost Breakdown per Minute of Conversation

Assuming ~50% of each minute is user speech and ~50% is agent speech:

| Component | Cloud API Cost (per min) | Notes |
|---|---|---|
| ASR | $0.005 - $0.025 | 30s of user speech. Deepgram Nova: $0.0058/min |
| LLM | $0.002 - $0.05 | ~200 tokens in, ~100 tokens out per turn, 3-5 turns/min |
| TTS | $0.005 - $0.04 | ~100 chars per turn, 3-5 turns/min |
| Transport (WebRTC) | $0.002 - $0.004 | Per-participant-minute |
| Telephony | $0.005 - $0.015 | PSTN termination costs |
| **Total** | **$0.02 - $0.13** | Depends heavily on provider choices |

### Optimization Strategies

**Reduce ASR cost:**
- Use VAD to only send speech segments to ASR. Background silence is free.
- Self-host Faster-Whisper or Distil-Whisper. GPU cost is fixed, not per-minute.
- Use cheaper ASR tiers for non-critical features (e.g., call summarization vs. real-time).

**Reduce LLM cost:**
- Use the smallest model that meets your quality bar. GPT-4o-mini and Gemini Flash are dramatically cheaper than frontier models.
- Trim context aggressively. Summarize earlier turns instead of passing full history.
- Cache common responses (greetings, FAQs, error messages).
- Self-host via vLLM when sustained volume justifies GPU cost.

**Reduce TTS cost:**
- Pre-generate static phrases ("How can I help you?", "One moment please.").
- Self-host Kokoro or Piper for high-volume deployments.
- Shorter agent responses = lower TTS cost + better user experience.

**Self-hosted cost crossover:**
- A single A100 GPU (~$2/hour on cloud) running Faster-Whisper + vLLM + Kokoro can serve ~20-50 concurrent sessions.
- At $0.10/min cloud cost and 50 concurrent sessions, cloud costs ~$300/hour. The A100 costs ~$2/hour. Self-hosting breaks even at very low utilization.
- But self-hosting requires engineering investment in deployment, scaling, monitoring, and on-call.

## Multilingual and Localization

Supporting multiple languages multiplies complexity across every layer of the voice stack. Each component may support different language sets and perform unevenly across them.

### Key Challenges

- **Code-switching** — Users mix languages mid-sentence ("I need to book a vuelo to Madrid"). Most ASR models handle this poorly.
- **Accent handling** — ASR accuracy drops significantly for non-native speakers and regional accents. Test with your actual user population.
- **Low-resource languages** — Smaller languages have fewer training data, worse ASR/TTS quality, and limited LLM support.
- **Cultural context** — Conversational norms vary. Pause length, formality, turn-taking patterns, and filler words differ across cultures.

### Multilingual ASR

| Tool | Languages | Multilingual Quality | Notes |
|---|---|---|---|
| Whisper Large V3 | 99 | Good for major languages, degrades for rare ones | Best open-source multilingual coverage |
| Deepgram Nova-3 | 36+ | Strong for supported languages | Separate multilingual pricing tier |
| Google Chirp 2 | 100+ | Consistently good across languages | Streaming support |
| Azure Speech | 100+ | Good | Custom model training per language |
| Moonshine | 8 | Good for supported languages | Edge-optimized |

### Multilingual TTS

| Tool | Languages | Quality | Notes |
|---|---|---|---|
| Cartesia Sonic | 42 | High | Voice localization feature to adapt voices across languages |
| ElevenLabs | 32 | Highest naturalness | Per-language voice selection |
| Azure Neural TTS | 140+ | Good | Widest language coverage |
| Kokoro | 5 | Good | English, French, Korean, Japanese, Mandarin |
| Piper | 30+ | Varies by language | Community-contributed voice packs |

### Tradeoffs

- **Single multilingual model vs. language-specific models** — A single model (Whisper) is simpler to deploy but accuracy varies by language. Per-language models (fine-tuned Parakeet for English, dedicated Mandarin ASR) are more accurate but multiply operational complexity.
- **Language detection** — Detect the user's language automatically in the first utterance and route to the appropriate pipeline. This adds latency to the first turn. Alternatively, let the user select their language upfront.
- **Voice consistency** — The same TTS voice in English may sound different (or not exist) in another language. Budget for per-language voice selection and testing.

## Compliance, Security, and Privacy

Voice AI systems process sensitive audio data and interact with humans in regulated contexts. Compliance requirements vary by jurisdiction and use case.

### Call Recording Laws

- **Two-party consent** — In many US states (California, Florida, Illinois) and most of the EU, all parties must consent to recording. Your voice agent must disclose recording at the start of the call.
- **One-party consent** — Some US states allow recording if one party (your AI agent) consents. Check your jurisdiction.
- **GDPR** — In the EU, voice data is personal data. You need a lawful basis for processing, must honor right-to-deletion requests, and need clear privacy notices.
- **HIPAA** — For healthcare voice AI, audio recordings and transcripts are Protected Health Information (PHI). Requires BAAs with all service providers.

### AI Disclosure

- Several US states now require disclosure that the caller is speaking with an AI (California, Colorado, Utah).
- The EU AI Act classifies conversational AI as a transparency-required system. Users must be informed they are interacting with AI.
- Best practice: always disclose upfront. Regulations are expanding, and user trust benefits from transparency.

### Data Security

- **Audio data in transit** — Use TLS/DTLS (WebRTC provides this by default). Never transmit raw audio over unencrypted channels.
- **Audio data at rest** — Encrypt stored recordings and transcripts. Implement access controls and audit logging.
- **PII in transcripts** — Transcripts may contain SSNs, credit card numbers, health information. Implement PII detection and redaction (AWS Comprehend, Presidio, or regex-based).
- **Third-party API exposure** — Every cloud ASR/LLM/TTS provider receives your user's data. Review their data processing agreements. Consider self-hosting for sensitive workloads.

### Retention and Deletion

- Define a retention policy for audio recordings, transcripts, and LLM conversation logs.
- Implement automated deletion pipelines.
- Respond to data subject access requests (DSARs) within regulatory timelines (30 days for GDPR).

### Security Best Practices

- Rate-limit voice AI endpoints to prevent abuse (toll fraud on telephony, prompt injection via audio).
- Validate and sanitize ASR transcripts before passing to LLMs to reduce prompt injection risk.
- Monitor for anomalous usage patterns (long calls, high-frequency calling, unusual geographic origins).
- Implement call recording consent mechanisms that are legally compliant in your operating jurisdictions.

## Contributing

Contributions are welcome. Please read the [contribution guidelines](contributing.md) before submitting a pull request.

## Footnotes

This list is maintained by the community. It is not affiliated with any of the listed tools or companies.

If you find an error or want to suggest an addition, please [open an issue](https://github.com/your-username/awesome-voice-stack/issues) or submit a pull request.

Licensed under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/).
