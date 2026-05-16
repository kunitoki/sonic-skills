# YUP Violation Catalogue

Full reference for `yup-review`. Each section shows the BAD pattern, why it is wrong, and the GOOD replacement.

---

## 1. Parameter Handle Patterns

### Using AudioParameterHandle before prepareToPlay

`yup::AudioParameterHandle` stores a raw parameter pointer and a smoother configured with the current sample rate. A default-constructed handle is not usable.

**BAD**
```cpp
class MyPlugin : public yup::AudioProcessor
{
    yup::AudioParameter::Ptr gainParam;
    yup::AudioParameterHandle gain;

    void processBlock (yup::AudioSampleBuffer& buffer, yup::MidiBuffer&) override
    {
        gain.updateNextAudioBlock(); // parameter pointer may still be null
        buffer.applyGain (gain.getCurrentValue());
    }
};
```

**GOOD**
```cpp
void prepareToPlay (float sampleRate, int maxBlockSize) override
{
    ignoreUnused (maxBlockSize);
    gain = yup::AudioParameterHandle (*gainParam, sampleRate);
}

void processBlock (yup::AudioSampleBuffer& buffer, yup::MidiBuffer&) override
{
    if (gain.updateNextAudioBlock())
    {
        const auto start = gain.getCurrentValue();
        const auto end = gain.skip (buffer.getNumSamples());
        buffer.applyGainRamp (0, buffer.getNumSamples(), start, end);
    }
    else
    {
        buffer.applyGain (gain.getCurrentValue());
    }
}
```

### Recreating handles in processBlock

Constructing the handle in the callback resets the smoother and makes automation zipper or stick.

**BAD**
```cpp
void processBlock (yup::AudioSampleBuffer& buffer, yup::MidiBuffer&) override
{
    yup::AudioParameterHandle gain (*gainParam, getSampleRate());
    gain.updateNextAudioBlock();
    buffer.applyGain (gain.getCurrentValue());
}
```

**GOOD**
Construct handles once in `prepareToPlay`, then update once per block.

---

## 2. Host Notification and Gestures

### setValueNotifyingHost from audio thread

`setValueNotifyingHost()` calls parameter listeners. Use it for UI/host-control paths, not for audio-rate processing.

**BAD**
```cpp
void processBlock (yup::AudioSampleBuffer&, yup::MidiBuffer& midi) override
{
    for (const auto metadata : midi)
        if (metadata.getMessage().isController())
            gainParam->setValueNotifyingHost (metadata.getMessage().getControllerValue() / 127.0f);
}
```

**GOOD**
```cpp
void processBlock (yup::AudioSampleBuffer&, yup::MidiBuffer& midi) override
{
    for (const auto metadata : midi)
        if (metadata.getMessage().isController())
            gainParam->setNormalizedValue (metadata.getMessage().getControllerValue() / 127.0f);
}
```

For editor controls, notify the host and bracket drags:

```cpp
slider->onDragStart = [this] (const yup::MouseEvent&) { gainParam->beginChangeGesture(); };
slider->onValueChanged = [this] (float value) { gainParam->setValueNotifyingHost (value); };
slider->onDragEnd = [this] (const yup::MouseEvent&) { gainParam->endChangeGesture(); };
```

---

## 3. Realtime Allocation and Logging

YUP core types such as `yup::String`, `yup::MemoryBlock`, file/network classes, JSON/XML helpers, `Logger`, and `YUP_DBG` can allocate or perform I/O. Keep them out of `processBlock` and DSP helpers called by it.

**BAD**
```cpp
void processBlock (yup::AudioSampleBuffer& buffer, yup::MidiBuffer&) override
{
    yup::String message = "samples=" + yup::String (buffer.getNumSamples());
    yup::Logger::writeToLog (message);
}
```

**GOOD**
```cpp
std::atomic<int> lastBlockSize { 0 };

void processBlock (yup::AudioSampleBuffer& buffer, yup::MidiBuffer&) override
{
    lastBlockSize.store (buffer.getNumSamples(), std::memory_order_relaxed);
}
```

The editor or a background diagnostic thread can format and log the value later.

---

## 4. MIDI and Voice Storage

### Allocating voices on note-on

Dynamic containers can grow during note-on bursts. Preallocate a bounded voice pool in `prepareToPlay`.

**BAD**
```cpp
void processBlock (yup::AudioSampleBuffer&, yup::MidiBuffer& midi) override
{
    for (const auto metadata : midi)
        if (metadata.getMessage().isNoteOn())
            voices.push_back (Voice { metadata.getMessage().getNoteNumber() }); // may allocate
}
```

**GOOD**
```cpp
static constexpr int maxVoices = 32;
std::array<Voice, maxVoices> voices {};

Voice* allocateVoice (int note)
{
    for (auto& voice : voices)
    {
        if (! voice.active)
        {
            voice.note = note;
            voice.active = true;
            voice.phase = 0.0f;
            return &voice;
        }
    }

    return stealQuietestVoice (note);
}
```

### Mutating MidiBuffer while iterating

Do not add/remove events from the same `MidiBuffer` iterator is traversing. Use a preallocated outgoing event buffer or collect bounded events, then rewrite after iteration.

**BAD**
```cpp
for (const auto metadata : midi)
    if (metadata.getMessage().isNoteOn())
        midi.addEvent (yup::MidiMessage::noteOff (1, metadata.getMessage().getNoteNumber()), 0);
```

**GOOD**
```cpp
std::array<yup::MidiMessage, 64> pendingNoteOffs {};
int pendingCount = 0;

for (const auto metadata : midi)
{
    if (metadata.getMessage().isNoteOn() && pendingCount < static_cast<int> (pendingNoteOffs.size()))
        pendingNoteOffs[pendingCount++] = yup::MidiMessage::noteOff (1, metadata.getMessage().getNoteNumber());
}

midi.clear();
for (int i = 0; i < pendingCount; ++i)
    midi.addEvent (pendingNoteOffs[i], 0);
```

---

## 5. Bus Layout and Channel Access

YUP exposes fixed bus layout through `yup::AudioBusLayout`, but `processBlock` still receives a buffer from the wrapper/host. Guard channel access unless the layout and format guarantee it.

**BAD**
```cpp
float* left = buffer.getWritePointer (0);
float* right = buffer.getWritePointer (1); // crashes for mono output
```

**GOOD**
```cpp
const int channels = buffer.getNumChannels();
const int samples = buffer.getNumSamples();

for (int ch = 0; ch < channels; ++ch)
{
    auto* out = buffer.getWritePointer (ch);
    for (int n = 0; n < samples; ++n)
        out[n] = renderSampleForChannel (ch);
}
```

If the algorithm requires stereo, declare stereo output in the processor constructor and validate that assumption in tests.

---

## 6. Lifecycle Reset and Flush

`prepareToPlay` can be called repeatedly on sample-rate or block-size changes. CLAP reset calls `flush()`. Both must leave the processor in a deterministic state.

Checklist:
- Reset all oscillators, filters, delay lines, envelopes, meters, FFT overlap buffers, and phase counters.
- Rebuild `AudioParameterHandle`s with the current sample rate.
- Clear or reinitialise voice state and pending MIDI.
- Resize scratch buffers to worst-case `maxBlockSize` outside the audio callback.
- In `flush()`, stop held voices and clear tails/pending events without freeing memory needed for the next block.

**BAD**
```cpp
void prepareToPlay (float sampleRate, int maxBlockSize) override
{
    ignoreUnused (sampleRate, maxBlockSize);
    gain = yup::AudioParameterHandle (*gainParam, sampleRate);
    // Filter history, delay buffers, held voices left untouched.
}
```

**GOOD**
```cpp
void prepareToPlay (float sampleRate, int maxBlockSize) override
{
    this->sampleRate = sampleRate;
    scratch.setSize (2, maxBlockSize);
    gain = yup::AudioParameterHandle (*gainParam, sampleRate);
    filter.reset();
    delay.clear();
    clearVoices();
}

void flush() override
{
    clearVoices();
    delay.clear();
    pendingMidi.clear();
}
```

---

## 7. State and Presets

Hosts rely on `loadStateFromMemory` and `saveStateIntoMemory` for session recall. Leaving them as `Result::fail ("Not implemented")` is only acceptable for throwaway examples.

**BAD**
```cpp
yup::Result saveStateIntoMemory (yup::MemoryBlock&) override
{
    return yup::Result::fail ("Not implemented");
}
```

**GOOD**
Serialise all parameter IDs/values and preset data into a versioned format, validate on load, then set parameters from the state callback:

```cpp
yup::Result loadStateFromMemory (const yup::MemoryBlock& data) override
{
    auto state = parseState (data);
    if (! state.valid)
        return yup::Result::fail ("Invalid plugin state");

    gainParam->setValue (state.gain);
    currentPreset = state.currentPreset;
    return yup::Result::ok();
}
```

If state restore swaps large shared DSP data, build it off the audio thread and publish it through a short lock-free or bounded handoff.

---

## 8. Recommended YUP Patterns Summary

| Concern | Recommended pattern |
|---------|---------------------|
| Parameter creation | `AudioParameterBuilder` in processor constructor; stable unique IDs |
| Audio-thread parameter reads | `AudioParameterHandle` initialised in `prepareToPlay`, updated once per block |
| UI parameter edits | `beginChangeGesture` / `setValueNotifyingHost` / `endChangeGesture` |
| Preset/state changes | `setValue` or `setNormalizedValue` from non-audio callbacks, then handle picks up change |
| Audio-to-UI meters | Atomics or preallocated SPSC queue; editor `Timer` polls |
| MIDI voice management | Preallocated bounded voice pool; no allocation on note-on |
| Bus handling | Explicit `AudioBusLayout`; channel-count guards in `processBlock` |
| Reset | Full reset in `prepareToPlay`; tail/voice clear in `flush()` |
