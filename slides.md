---
theme: default
colorSchema: light
title: Type-Erased Audio Parameters
info: |
  ## Type-Erased Audio Parameters

  Jan Wilczek's Audio Developer Conference Japan 2026 talk
author: Jan Wilczek
# apply UnoCSS classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable Comark Syntax: https://comark.dev/syntax/markdown
comark: true
# duration of the presentation
duration: 35min
---

# Type-Erased Audio Parameters

## A New Approach to an Old Problem

Jan Wilczek (WolfSound)

ADC Japan 2026

---

# Who am I?

- Jan Wilczek \[Yan Vil-check\]
- Audio programming consultant & educator
- Founder of TheWolfSound.com blog & YouTube channel on audio programming
- WolfTalk podcast host
- Trainer
    - conference workshops
    - in-house training on DSP/JUCE
- Online course creator
    - DSP Pro on digital audio signal processing
    - Official JUCE C++ framework audio plugin development course (over 4,400 students enrolled)

---

# Who here develops or uses audio plugins for digital audio workstations?

<!-- either professionally or as a hobby? -->

---

# Who here uses JUCE to develop plugins?

<!-- Well, let me tell you the story of developing my plugin -->

---

# The Story of a Synth

---

# Plugin processor

```cpp {all|40}
class EdenSynthAudioProcessor : public AudioProcessor {
public:
  EdenSynthAudioProcessor();
  ~EdenSynthAudioProcessor();

  void prepareToPlay(double sampleRate, int samplesPerBlock) override;
  void releaseResources() override;

#ifndef JucePlugin_PreferredChannelConfigurations
  bool isBusesLayoutSupported(const BusesLayout& layouts) const override;
#endif

  void processBlock(AudioBuffer<float>&, MidiBuffer&) override;

  AudioProcessorEditor* createEditor() override;
  bool hasEditor() const override;

  const String getName() const override;

  bool acceptsMidi() const override;
  bool producesMidi() const override;
  bool isMidiEffect() const override;
  double getTailLengthSeconds() const override;

  int getNumPrograms() override;
  int getCurrentProgram() override;
  void setCurrentProgram(int index) override;
  const String getProgramName(int index) override;
  void changeProgramName(int index, const String& newName) override;

  void getStateInformation(MemoryBlock& destData) override;
  void setStateInformation(const void* data, int sizeInBytes) override;

private:
  JUCE_DECLARE_NON_COPYABLE_WITH_LEAK_DETECTOR(EdenSynthAudioProcessor)

  std::filesystem::path _assetsPath;
  eden::EdenSynthesiser _edenSynthesiser;
  eden_vst::EdenAdapter _edenAdapter;
  AudioProcessorValueTreeState _pluginParameters;
};
```

---

# Parameters via `AudioProcessorValueTreeState`

```cpp {all|3|4-16|18}
EdenSynthAudioProcessor::EdenSynthAudioProcessor()
    : //...
      _pluginParameters(*this, nullptr) {
  using Parameter = juce::AudioProcessorValueTreeState::Parameter;

  _pluginParameters.createAndAddParameter(std::make_unique<Parameter>(
      "pitchBend.semitonesDown", "Pitch bend semitones down",
      NormalisableRange<float>(-24.f, 0.f, 1.f), -12.f));
  _pluginParameters.createAndAddParameter(std::make_unique<Parameter>(
      "pitchBend.semitonesUp", "Pitch bend semitones up",
      NormalisableRange<float>(0.f, 24.f, 1.f), 2.f));
  _pluginParameters.createAndAddParameter(std::make_unique<Parameter>(
      "frequencyOfA4", "Frequency of A4",
      NormalisableRange<float>(400.f, 500.f, 0.1f), 440.f,
      AudioProcessorValueTreeStateParameterAttributes{}.withLabel("Hz")));
  // more parameters...

  _pluginParameters.state = ValueTree(Identifier("EdenSynthParameters"));
}
```

 <!-- `createAndAddParameter()` API is deprecated (?) -->

---

# Parameters via `AudioProcessorValueTreeState`

```cpp
void EdenSynthAudioProcessor::processBlock(AudioBuffer<float>& buffer,
                                           MidiBuffer& midiMessages) {
  //...

  _synthesiser.setPitchBendRange(
      {static_cast<int>(
           *pluginParameters.getRawParameterValue("pitchBend.semitonesDown")),
       static_cast<int>(
           *pluginParameters.getRawParameterValue("pitchBend.semitonesUp"))});
  _synthesiser.setFrequencyOfA4(
      *pluginParameters.getRawParameterValue("frequencyOfA4"));

  // audio & MIDI processing
}
```

---

# Parameters via `AudioProcessorValueTreeState`

```cpp
void EdenSynthAudioProcessor::getStateInformation(MemoryBlock& destData) {
  auto state = _pluginParameters.copyState();
  const std::unique_ptr<XmlElement> xml(state.createXml());
  copyXmlToBinary(*xml, destData);
}
```

---

# Parameters via `AudioProcessorValueTreeState`

```cpp
void EdenSynthAudioProcessor::setStateInformation(const void* data,
                                                  int sizeInBytes) {
  std::unique_ptr<XmlElement> xmlState(getXmlFromBinary(data, sizeInBytes));

  if (xmlState.get()) {
    if (xmlState->hasTagName(_pluginParameters.state.getType())) {
      _pluginParameters.replaceState(ValueTree::fromXml(*xmlState));
    }
  }
}
```

---

```cpp {all|3|11-12|5}
class GeneralSettingsComponent : public Component {
public:
  using SliderAttachment = AudioProcessorValueTreeState::SliderAttachment;

  GeneralSettingsComponent(AudioProcessorValueTreeState&);

  void resized() override;
  void paint(Graphics& g) override;

private:
  Slider _pitchBendSemitonesUp;
  std::unique_ptr<SliderAttachment> _pitchBendSemitonesUpAttachment;

  Slider _pitchBendSemitonesDown;
  std::unique_ptr<SliderAttachment> _pitchBendSemitonesDownAttachment;

  Slider _a4Frequency;
  std::unique_ptr<SliderAttachment> _a4FrequencyAttachment;
};
```

---

```cpp {all|7-8}
GeneralSettingsComponent::GeneralSettingsComponent(
    AudioProcessorValueTreeState& valueTreeState)
    : _pitchBendSemitonesUp{/* */},
      _pitchBendSemitonesDown{/* */},
      _a4Frequency{/* */} {
  addAndMakeVisible(_pitchBendSemitonesUp);
  _pitchBendSemitonesUpAttachment = std::make_unique<SliderAttachment>(
      valueTreeState, "pitchBend.semitonesUp", _pitchBendSemitonesUp);

  addAndMakeVisible(_pitchBendSemitonesDown);
  _pitchBendSemitonesDownAttachment = std::make_unique<SliderAttachment>(
      valueTreeState, "pitchBend.semitonesDown", _pitchBendSemitonesDown);

  addAndMakeVisible(_a4Frequency);
  _a4FrequencyAttachment = std::make_unique<SliderAttachment>(
      valueTreeState, "frequencyOfA4", _a4Frequency);
}
```

---

# How to add presets? 🤔

---

# Reuse `get/setStateInformation()`

```cpp
void savePreset(const std::string& presetName) {
    juce::MemoryBlock presetData;
    pluginProcessor.getStateInformation(result);
    const auto presetFile = juce::File{presetPathFromName(name)};
    presetFile.deleteFile();
    presetFile.appendData(presetData.getData(), presetData.getSize());
}
```

---

# Reuse `get/setStateInformation()`

```cpp
std::expected<PresetLoadingSuccess, PresetLoadingError> loadPreset(const std::string& presetName) {
  const auto presetFile = juce::File{presetPathFromName(name)};

  if (!presetFile.existsAsFile()) {
    return std::unexpected{PresetLoadingError::DoesNotExist};
  }

  if (!presetFile.hasReadAccess()) {
    return std::unexpected{PresetLoadingError::NoPermission};
  }

  juce::MemoryBlock presetData;
  const auto result = presetFile.loadFileAsData(presetData);

  if (!result) {
    return std::unexpected{PresetLoadingError::FailedToReadFile};
  }

  pluginProcessor.setStateInformation(presetData.getData(),
                                      static_cast<int>(presetData.getSize()));

  return PresetLoadingSuccess::Ok;
}
```

---



---

# Type-Erased Parameters (bottom-up)

---

## ParameterWrapper

```cpp
template <class Parameter>
class ParameterWrapper {
public:
    ParameterWrapper(Parameter& p) : _p{p} {}

private:
    Parameter& _p;
};
```

---

## ParameterWrapper

```cpp
template <class Parameter>
class ParameterWrapper {
public:
    ParameterWrapper(Parameter& p) : _p{p} {}



private:
    Parameter& _p;
};
```

---

# Type-Erased Parameters (top-down)

---

# Collection of parameters

```cpp
class TypeErasedParameter;

std::vector<TypeErasedParameter> parameters;
```

---

# `TypeErasedParameter`

```cpp
class TypeErasedParameter {
public:
    TypeErasedParameter(juce::AudioParameterFloat& p) : _p{p} {}

private:
    juce::AudioParameterFloat& _p;
};
```

---

# `TypeErasedParameter`

```cpp
class TypeErasedParameter {
public:
    template <class Parameter>
    TypeErasedParameter(Parameter& p) : _p{p} {}

private:
    Parameter& _p;
};
```

---

# `TypeErasedParameter`

```cpp
template <class Parameter>
class TypeErasedParameter {
public:
    TypeErasedParameter(Parameter& p) : _p{p} {}

private:
    Parameter& _p;
};
```

---

# `TypeErasedParameter`

```cpp
template <class Parameter>
class TypeErasedParameter {
public:
    TypeErasedParameter(Parameter& p) : _p{p} {}

private:
    Parameter& _p;
};

std::vector<TypeErasedParameter<?>> parameters;
```

---

# `TypeErasedParameter`

```cpp
class TypeErasedParameter {
public:
    template <class Parameter>
    TypeErasedParameter(Parameter& p) : _impl{std::make_unique<ParameterModel<Parameter>>(p)} {}

private:
    template <class Parameter>
    class ParameterModel {
    public:
        ParameterModel(Parameter& p) : _p{p} {}
    private:
        Parameter& _p;
    };

    std::unique_ptr<ParameterModel<?>> _impl; // <- problem
};
```

---

# `TypeErasedParameter`

```cpp {all|17}
class TypeErasedParameter {
public:
    template <class Parameter>
    TypeErasedParameter(Parameter& p) : _impl{std::make_unique<ParameterModel<Parameter>>(p)} {}

private:
    class ParameterConcept {
    public:
        virtual ~ParameterConcept() = default;
    };

    template <class Parameter>
    class ParameterModel : public ParameterConcept {
    public:
        ParameterModel(Parameter& p) : _p{p} {}
    private:
        Parameter& _p;
    };

    std::unique_ptr<ParameterConcept> _impl;
};
```

---

# `TypeErasedParameter`

```cpp {17|all|23}
class TypeErasedParameter {
public:
    template <class Parameter>
    TypeErasedParameter(Parameter& p) : _impl{std::make_unique<ParameterModel<Parameter>>(p)} {}

private:
    class ParameterConcept {
    public:
        virtual ~ParameterConcept() = default;
    };

    template <class Parameter>
    class ParameterModel : public ParameterConcept {
    public:
        ParameterModel(Parameter& p) : _p{p} {}
    private:
        std::reference_wrapper<Parameter> _p; // copyable
    };

    std::unique_ptr<ParameterConcept> _impl;
};

std::vector<TypeErasedParameter> parameters;
```


<!-- Nice! We have our TypeErasedParameter, but what have achieved? Well, we can now store parameters of arbitrary types in a vector. We don't use any hacks, we don't use the pointer to base in the public-facing API, and we are entirely type-safe. Now, we want to make useful operations on the parameters; how?  -->

---

# Operations

```cpp {1-3,7,12,18}
void foo(juce::AudioParameterFloat& p);
void foo(juce::AudioParameterBool& p);
//...
class TypeErasedParameter {
public:
    //...
    void foo() { _impl->foo(); }
private:
    class ParameterConcept {
    public:
        virtual ~ParameterConcept() = default;
        virtual void foo() = 0;
    };
    template <class Parameter>
    class ParameterModel : public ParameterConcept {
    public:
        //...
        void foo() override { foo(_p); }

    private:
        std::reference_wrapper<Parameter> _p;
    };
    std::unique_ptr<ParameterConcept> _impl;
};
```

---

# Operations

```cpp {6,12,20}
class TypeErasedParameter {
public:
    template <class Parameter>
    TypeErasedParameter(Parameter& p) : _impl{std::make_unique<ParameterModel<Parameter>>(p)} {}

    float getValue() { return _impl->getValue(); }

private:
    class ParameterConcept {
    public:
        virtual ~ParameterConcept() = default;
        virtual float getValue() = 0;
    };

    template <class Parameter>
    class ParameterModel : public ParameterConcept {
    public:
        ParameterModel(Parameter& p) : _p{p} {}

        float getValue() override { return _p.get().getValue(); } // value in [0,1] range

    private:
        std::reference_wrapper<Parameter> _p;
    };

    std::unique_ptr<ParameterConcept> _impl;
};
```

<!-- No type safety! -->

---

# Operations

```cpp {4,10,18}
class TypeErasedParameter {
public:
    //...
    ??? getValue() { return _impl->getValue(); }

private:
    class ParameterConcept {
    public:
        virtual ~ParameterConcept() = default;
        virtual ??? getValue() = 0;
    };

    template <class Parameter>
    class ParameterModel : public ParameterConcept {
    public:
        ParameterModel(Parameter& p) : _p{p} {}

        ??? getValue() override { return _p.get().get(); } // returns bool, float, etc.

    private:
        std::reference_wrapper<Parameter> _p;
    };

    std::unique_ptr<ParameterConcept> _impl;
};
```

---

# Operations

```cpp {4-5,12-13}
class TypeErasedParameter {
public:
    //...
    template <typename Value>
    void getValue(Value& v) { _impl->getValue(v); }

private:
    class ParameterConcept {
    public:
        virtual ~ParameterConcept() = default;

        template <class Value>
        virtual void getValue(Value& v) = 0; // invalid
    };

    //...
};
```

<!-- There is no way around it: we need a concrete type to read out the value, period. -->

---

# Operations

