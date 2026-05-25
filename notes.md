## Things to say

- What are parameters, where are they used in audio plugins
- JUCE API requirements on parameters
    - addParameter()
    - get/setSerialisedState()
    - manipulation:
        - retain a pointer/reference
        - call getParameter() (?) -> the type is lost
    - set on DSP in processBlock()
- Changes in JUCE 9
    - APv2 has not yet been published, so I cannot discuss the changes therein
- Possible aid: APVTS
    - name is too long!
    - some things wrong with it
    - how to use it
        - dynamic_cast<>()
        - addToLayout() from Attila
- The general problem: We need access to singular parameters ideallly as strong types (for get/set and nice display (?), e.g., AudioParameterChoice)
- AudioParameter* classes in JUCE
- What's interesting, GenericAudioProcessorEditor, uses hacks to determine parameter type

## Submitted Outline

1. Introduction
2. Plugin parameters, their use cases, features, and challenges
	1. Overview of `juce::AudioParameter*` classes
	2. Value labelling and formatting
	3. Observability
3. Criteria for a desired plugin parameter solution
	1. Integration with the `juce::AudioProcessor::addParameter()` API
	2. Type safety (no raw `float`s, no `dynamic_cast`s, type-safe access)
	3. Treating parameters as a collection
	4. Easy, minimal-code serialization
	5. Preset support
	6. Minimal code on the client side
	7. Support for parameter groups and meta parameters
4. Existing solutions to maintaining plugin parameters in a codebase and their problems
	1. `juce::AudioProcessorValueTreeState`
	2. `addToLayout<T>()` technique
5. Developing a type-erased plugin parameter system
---
6. Critique of the type-erasure solution
7. Guideline on transitioning to the type-erased parameter system in existing codebases
8. Open-source implementation example
9. Discussion
    1. As usual in software engineering, there isn't a single perfect solution. A critique of the proposed approach from the audience should help build a better understanding of the problem and possible solutions among the participants, and provide additional value.

## Hacks in GenericAudioProcessorEditor

```
std::unique_ptr<ParameterComponent> createParameterComp (AudioProcessor& processor) const
{
    // The AU, AUv3 and VST (only via a .vstxml file) SDKs support
    // marking a parameter as boolean. If you want consistency across
    // all  formats then it might be best to use a
    // SwitchParameterComponent instead.
    if (parameter.isBoolean())
        return std::make_unique<BooleanParameterComponent> (processor, parameter);

    // Most hosts display any parameter with just two steps as a switch.
    if (parameter.getNumSteps() == 2)
        return std::make_unique<SwitchParameterComponent> (processor, parameter);

    // If we have a list of strings to represent the different states a
    // parameter can be in then we should present a dropdown allowing a
    // user to pick one of them.
    // We limit the number of items in the dropdown to avoid having to
    // build and display an overly-large menu.
    constexpr auto arbitraryDiscreteChoiceThreshold = 1000;

    if (const auto numSteps = parameter.getNumSteps();
        numSteps < arbitraryDiscreteChoiceThreshold)
    {
        const auto valueStrings = parameter.getAllValueStrings();

        if (! valueStrings.isEmpty() && std::abs (numSteps - valueStrings.size()) <= 1)
            return std::make_unique<ChoiceParameterComponent> (processor, parameter);
    }

    // Everything else can be represented as a slider.
    return std::make_unique<SliderParameterComponent> (processor, parameter);
}
```
