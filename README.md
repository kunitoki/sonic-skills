![Sonic Skills](backdrop.jpg)

# Sonic Skills

Precision audio-engineering skills for AI agents.

Sonic Skills is a curated pack of Markdown skills for reviewing, debugging, explaining, and implementing audio software. The focus is practical: realtime safety, DSP correctness, plugin host compatibility, Web Audio behavior, and the kinds of edge cases that turn into clicks, dropouts, broken recall, or unstable filters.

## What It Covers

| Area | Skills | Use for |
|---|---|---|
| Core DSP safety | `audio-dsp-review`, `audio-numerics-review`, `audio-performance-debug` | Realtime violations, floating-point traps, denormals, xruns, profiling strategy |
| Debugging and explanation | `audio-artifacts-debug`, `audio-signal-flow-explainer`, `audio-math-explainer` | Clicks, aliasing, DC offset, routing diagrams, DSP intuition |
| DSP implementation | `dsp-algorithm-guide` | Biquads, SVFs, compressors, FDN reverbs, FFT convolution, phase vocoders, PolyBLEP |
| Plugin formats | `vst3-review`, `clap-review`, `au-review` | Spec compliance, host contracts, parameter/state handling, render-thread rules |
| Frameworks | `juce-guide`, `juce-review`, `yup-guide`, `yup-review` | JUCE and YUP plugin setup, parameters, UI/thread boundaries, MIDI and buffer safety |
| Web audio | `webaudio-guide`, `webaudio-review` | AudioWorklet processors, graph construction, automation, WebMIDI, browser constraints |
| Game audio | `game-audio-guide`, `game-audio-review` | Wwise/FMOD DSP plugins, middleware allocators, RTPC/parameter patterns, engine thread safety |

## How To Use

Point your agent at this repository and ask for the skill by name, or describe the task naturally.

Examples:

```text
/audio-dsp-review inspect the current project
```

```text
/webaudio-review why does this AudioWorklet crackle under load?
```

```text
/dsp-algorithm-guide implement a stable state variable lowpass filter
```

```text
/clap-review inspect this CLAP plugin's process() and params extension
```

Skills are designed to compose. A VST3 review, for example, should usually start with `audio-dsp-review` and `audio-numerics-review`, then add `vst3-review` for format-specific contracts.

## Skill Catalog

| Skill | Purpose |
|---|---|
| [`audio-dsp-review`](skills/audio-dsp-review/SKILL.md) | Finds allocations, locks, I/O, system calls, and other audio-thread hazards. |
| [`audio-numerics-review`](skills/audio-numerics-review/SKILL.md) | Finds denormals, NaN/Inf propagation, cancellation, fixed-point overflow, and precision loss. |
| [`audio-performance-debug`](skills/audio-performance-debug/SKILL.md) | Gives profiling workflows and optimization patterns for xruns, spikes, and high CPU. |
| [`audio-artifacts-debug`](skills/audio-artifacts-debug/SKILL.md) | Maps audible symptoms to likely root causes and targeted checks. |
| [`audio-signal-flow-explainer`](skills/audio-signal-flow-explainer/SKILL.md) | Traces sources, processors, routers, sidechains, feedback, latency, and sinks. |
| [`audio-math-explainer`](skills/audio-math-explainer/SKILL.md) | Explains DSP concepts with intuition, formula, and code connection. |
| [`dsp-algorithm-guide`](skills/dsp-algorithm-guide/SKILL.md) | Guides implementation and verification of common DSP blocks. |
| [`juce-guide`](skills/juce-guide/SKILL.md) | Walks through building a JUCE plugin with CMake, APVTS, UI, state, and packaging. |
| [`juce-review`](skills/juce-review/SKILL.md) | Reviews JUCE-specific thread boundaries, APVTS, ValueTree, MessageManager, MIDI, and buffers. |
| [`yup-guide`](skills/yup-guide/SKILL.md) | Walks through building a YUP plugin with `yup_audio_plugin`, `AudioProcessor`, `AudioParameterHandle`, UI, state, and packaging. |
| [`yup-review`](skills/yup-review/SKILL.md) | Reviews YUP-specific processor lifecycle, parameter handles, editor gestures, bus layout, state recall, and wrapper contracts. |
| [`vst3-review`](skills/vst3-review/SKILL.md) | Reviews VST3 bus negotiation, processor/controller separation, parameters, state, latency, and tail behavior. |
| [`clap-review`](skills/clap-review/SKILL.md) | Reviews CLAP threading tags, event lifetime, params/state extensions, ports, and process status. |
| [`au-review`](skills/au-review/SKILL.md) | Reviews AUv2/AUv3 render blocks, properties, parameters, tail time, sandboxing, and `auval` readiness. |
| [`webaudio-guide`](skills/webaudio-guide/SKILL.md) | Guides Web Audio pipelines, AudioWorklets, automation, worklet communication, and WebMIDI. |
| [`webaudio-review`](skills/webaudio-review/SKILL.md) | Reviews AudioWorklet safety, deprecated APIs, browser policies, and shared-memory patterns. |
| [`game-audio-guide`](skills/game-audio-guide/SKILL.md) | Guides Wwise/FMOD custom DSP plugin setup and callback structure. |
| [`game-audio-review`](skills/game-audio-review/SKILL.md) | Reviews Wwise/FMOD/Unreal middleware integration and game/audio-thread safety. |

## Repository Layout

```text
skills/
  <skill-name>/
    SKILL.md              # trigger metadata and core workflow
    references/           # optional deeper checklists, patterns, formulas, examples
```

Each `SKILL.md` is intentionally compact. Larger code patterns and spec checklists live under `references/` so agents can load them only when needed.

## Design Principles

- **Realtime first:** the audio callback is a deadline, not a suggestion.
- **Spec-aware:** plugin-format advice is grounded in host/API contracts, not folklore.
- **Practical over academic:** formulas are connected to code paths, parameter ranges, and tests.
- **Composable:** universal DSP checks pair with framework- or format-specific reviews.
- **Evidence-oriented:** good reviews include file/line findings, why the issue matters, and concrete fixes.

## Contributing

When adding or editing a skill:

1. Keep `SKILL.md` focused on workflow and decision points.
2. Move detailed examples, tables, and long references into `references/`.
3. Validate technical claims against primary docs, specifications, or papers.
4. Prefer actionable review language: symptom, cause, risk, fix.
5. Avoid broad rules that are only sometimes true. State the condition.

## License

See [`LICENSE`](LICENSE).
