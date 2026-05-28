# Outline

1. who am I 
    - audio developer
    - creator of TheWolfSound.com blog and YouTube channel
    - WolfTalk podcast host
    - creator of DSP Pro and official JUCE online courses
    - audio programming trainer
        - I can help you with in-house training
1. Who here develops or uses audio plugins for digital audio workstations, either professionally or as a hobby?
1. Let me tell you the story of my first synthesizer
    - "If you don't get everything here, don't worry"
    - My first plugin ever was a moderately advanced sound synthesizer with multiple modules
    - It had quite a lot of parameters
        - show GUI
        - name some of the parameters the controls represent
    - Using APVTS according to JUCE tutorials to handle parameters and persistence
    - Wanting to add presets
        - problem with weak types (?)
        - lack of control of output (XML-only while I wanted a JSON)
1. What are parameters
    - means to control our audio processing algorithm
    - primitive values (float, int, bool, enum) + name + allowed range + minor features (unit, skew)
        - values in [0, 1] range for some formats ? (VST3?)
    - allow for
        - changing the processing of our plugin in real time
        - observing for value change
        - parameter automation
        - automatic preset creation (e.g., in Reaper)
        - plugin state persistence get/setSerializedState
        - creating a generic plugin editor
            - Reaper
            - JUCE (GenericAudioProcessorEditor)
        - creating a custom plugin editor
        - parameter labelling and formatting
        - DSP algorithm state visualization
    - the lifetime of parameters is bound to the audio processor
    - challenge: they must be real-time-safe (solved in practice by lock-free atomic types)
1. Parameter API in JUCE
    1. AudioParameter* classes
    1. Inheritance hierarchy
1. Parameter lifecycle
    1. Instantiate dynamically
    1. Pass an owning pointer to addParameter()
    1. Now, how to use in processBlock()?
        - we need a pointer/reference
        - we can call AudioProcessor::getParameter() (?) to obtain the parameter, but we lose the type information
    1. We can create UI controls for it
    1. We must serialize them in get/setSerialisedState(), otherwise, the plugin won't "remember" parameter values
1. The dominant approach today: AudioProcessorValueTreeState
    - APVTS
    - Not AudioParameterValueTreeState
    - Goal: managing a parameter collection + automatic UI/processor sync + serialization
    - How to use it
        - adding parameters
        - adding to processor
        - attaching UI widgets
        - serialization
        - preset support (?)
    - Problems with APVTS
        - the name
        - lack of type safety (?)
        - lost type of parameters (unless you do a dynamic_cast<>() or addToLayout<>() trick by Attila -> show both)
            - problem with dynamic_cast<> and pointers
                - pointers can be null
                - dynamic_cast<> is verbose and error-prone (duplication)
                - the code becomes overly verbose and imperative
        - no possibility to specify the output format
        - possibility of a deadlock -> don't ask me, ask JUCE experts
        - other issues (parameter groups)
1. An alternative: how we did it in the official JUCE course
    - Use a generic addParameter<>() function
    - Retain a reference
    - Use ParameterAttachment classes in the UI
    - Problem: no automatic serialization/deserialization
        - I don't like the idea of having to remember to add each new parameter into serialization/deserialization code. There aren't any mechanism to fail compilation if we add a parameter but forget to serialize it. Maybe C++ 26 reflection could help here, but we are a long way from having it in all major compilers.
1. Short summary: we want to have strongly typed parameters (for updating the DSP algorithm and serialization), but we also want to have a way of performing an operation on ALL plugin parameters, ideally preserving the parameter type. In other words, we want to treat the parameters as a collection of objects of different types.
1. Criteria for a desired plugin parameter solution
    1. Integration with the `juce::AudioProcessor::addParameter()` API
    2. Type safety (no raw `float`s, no `dynamic_cast`s, type-safe access)
    3. Treating parameters as a collection
    4. Easy, minimal-code serialization
    5. Preset support
    6. Minimal code on the client side
    7. Support for parameter groups and meta parameters
1. Developing a type-erased plugin parameter system
    1. I won't explain now what type erasure is; instead let's focus on developing such a solution
    1. I want to avoid virtual polymorphism and not store the parameters via a pointer to base, because that makes us lose the type information (we would need to use dynamic_cast to retreive it)
    1. I don't want to subclass all juce::AudioParameter* classes just to customize their serialization, because that seems like a lot of work resulting in a very unstable solution. We also don't want to reimplement these classes; I, just as probably you, already use JUCE parameter classes and I wouldn't like to rewrite this whole system, just to add flexible serialization.
    1. We want to have a collection. Let's represent this collection as a vector of `TypeErasedAudioParameter`s. `TypeErasedAudioParameter` is a class with value semantics that somehow wraps `juce::AudioParameter*` class.
    1. `TypeErasedAudioParameter` structure (TODO)

