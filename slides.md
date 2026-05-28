---
theme: default
title: Welcome to Slidev
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
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

# Welcome to Slidev

Presentation slides for developers

---

# What is Slidev?

Slidev is a slides maker and presenter designed for developers, consist of the following features

- 📝 **Text-based** - focus on the content with Markdown, and then style them later
- 🎨 **Themable** - themes can be shared and re-used as npm packages
- 🧑‍💻 **Developer Friendly** - code highlighting, live coding with autocompletion
- 🤹 **Interactive** - embed Vue components to enhance your expressions
- 🎥 **Recording** - built-in recording and camera view
- 📤 **Portable** - export to PDF, PPTX, PNGs, or even a hostable SPA
- 🛠 **Hackable** - virtually anything that's possible on a webpage is possible in Slidev
<br>
<br>

Read more about [Why Slidev?](https://sli.dev/guide/why)

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

