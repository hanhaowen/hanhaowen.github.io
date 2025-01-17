---
layout:     post   				   
title:      The Principle of Operator Registration in PyTorch		
subtitle:   
date:       2023-12-21		
author:     HanHaowen 			
header-img: img/post-bg-desk.jpg 	
catalog: false 						
tags:								
    - CUDA
    - Artificial Intelligence
    - C++
    - Pytorch
---
Note: This is a beginner's article, feedback is welcome! The following content is based on PyTorch 2.0.0.

In the official PyTorch tutorial [Extend Dispatcher](https://pytorch.org/tutorials/advanced/extend_dispatcher.html), the main way to register operators is explained as follows:

```cpp
TORCH_LIBRARY_IMPL(aten, AutogradPrivateUse1, m) {
  m.impl(<myadd_schema>, &myadd_autograd);
}
```

It is important to note that in C/C++, `&function` and `function` are the same, as seen in this [Stack Overflow post](https://stackoverflow.com/questions/6293403/in-c-what-is-the-difference-between-function-and-function-when-passed-as-a).

In the PyTorch code, the `TORCH_LIBRARY_IMPL` macro is defined in `/home/pytorch/torch/library.h` as follows:

```cpp
#define TORCH_LIBRARY_IMPL(ns, k, m) _TORCH_LIBRARY_IMPL(ns, k, m, C10_UID)
```

The `_TORCH_LIBRARY_IMPL` macro is defined as:

```cpp
#define _TORCH_LIBRARY_IMPL(ns, k, m, uid)                             \
  static void C10_CONCATENATE(                                         \
      TORCH_LIBRARY_IMPL_init_##ns##_##k##_, uid)(torch::Library&);    \
  static const torch::detail::TorchLibraryInit C10_CONCATENATE(        \
      TORCH_LIBRARY_IMPL_static_init_##ns##_##k##_, uid)(              \
      torch::Library::IMPL,                                            \
      c10::guts::if_constexpr<c10::impl::dispatch_key_allowlist_check( \
          c10::DispatchKey::k)>(                                       \
          []() {                                                       \
            return &C10_CONCATENATE(                                   \
                TORCH_LIBRARY_IMPL_init_##ns##_##k##_, uid);           \
          },                                                           \
          []() { return [](torch::Library&) -> void {}; }),            \
      #ns,                                                             \
      c10::make_optional(c10::DispatchKey::k),                         \
      __FILE__,                                                        \
      __LINE__);                                                       \
  void C10_CONCATENATE(                                                \
      TORCH_LIBRARY_IMPL_init_##ns##_##k##_, uid)(torch::Library & m)
```

Let's start by looking at `C10_UID`, which is defined as:

```cpp
#define C10_UID __COUNTER__
#define C10_ANONYMOUS_VARIABLE(str) C10_CONCATENATE(str, __COUNTER__)
```

Therefore, `C10_UID` is actually a globally unique ID number generated by the `__COUNTER__` macro.

The definition of `C10_CONCATENATE` is as follows:

```cpp
#define C10_CONCATENATE_IMPL(s1, s2) s1##s2
#define C10_CONCATENATE(s1, s2) C10_CONCATENATE_IMPL(s1, s2)
```

As you can see, it concatenates two strings together. If you're not familiar with it, you can look up the usage of the `##` operator in C/C++ preprocessing.

The definition of `_TORCH_LIBRARY_IMPL` can be divided into three parts:

1. Declaration of a static function:
```cpp
static void C10_CONCATENATE(TORCH_LIBRARY_IMPL_init_##ns##_##k##_, uid)(torch::Library&);
```
The function name is `TORCH_LIBRARY_IMPL_init_+ns+k+uid`. Assuming the UID for `TORCH_LIBRARY_IMPL(aten, AutogradPrivateUse1, m)` is 20, the function name would be:
`TORCH_LIBRARY_IMPL_init_aten_AutogradPrivateUse1_20`.

2. Definition of a constant within the cpp file:
```cpp
static const torch::detail::TorchLibraryInit C10_CONCATENATE(        \
    TORCH_LIBRARY_IMPL_static_init_##ns##_##k##_, uid)(              \
    torch::Library::IMPL,                                            \
    c10::guts::if_constexpr<c10::impl::dispatch_key_allowlist_check( \
        c10::DispatchKey::k)>(                                       \
        []() {                                                       \
          return &C10_CONCATENATE(                                   \
              TORCH_LIBRARY_IMPL_init_##ns##_##k##_, uid);           \
        },                                                           \
        []() { return [](torch::Library&) -> void {}; }),            \
    #ns,                                                             \
    c10::make_optional(c10::DispatchKey::k),                         \
    __FILE__,                                                        \
    __LINE__);                                                       \
```
The constant is of type `static const torch::detail::TorchLibraryInit`. Using the previous example, its name would be:
`TORCH_LIBRARY_IMPL_static_init_aten_AutogradPrivateUse1_20`. The only difference between this and the previously defined static function name is the addition of the "static" string. After macro expansion, the entire code segment would be as follows:
```cpp
static const torch::detail::TorchLibraryInit                     // Return type
TORCH_LIBRARY_IMPL_static_init_aten_AutogradPrivateUse1_20(          
    torch::Library::IMPL,                                        // Parameter 1, Library::Kind type    
    c10::guts::if_constexpr<c10::impl::dispatch_key_allowlist_check(c10::DispatchKey::AutogradPrivateUse1)>(                                       
        []() {return &TORCH_LIBRARY_IMPL_init_aten_AutogradPrivateUse1_20;},           
        []() { return [](torch::Library&) -> void {}; }
        ),                                                      // Parameter 2, InitFn* type
    "aten",                                                     // Parameter 3, const char* type 
    c10::make_optional(c10::DispatchKey::AutogradPrivateUse1),  // Parameter 4, c10::optional<c10::DispatchKey> type          
    __FILE__,                                                   // Parameter 5, const char* type
    __LINE__);                                                  // Parameter 6, uint32_t type
```

The class definition of `TorchLibraryInit` is as follows:
```cpp
class TorchLibraryInit final {
 private:
  using InitFn = void(Library&);
  Library lib_;

 public:
  TorchLibraryInit(
      Library::Kind kind,
      InitFn* fn,
      const char* ns,
      c10::optional<c10::DispatchKey> k,
      const char* file,
      uint32_t line)
      : lib_(kind, ns, k, file, line) {
    fn(lib_);
  }
};
```
It has a private member variable that is of type `Library`. In the constructor, it first initializes `lib_` with `kind, ns, k, file, line`, and then initializes the private member variable `lib_` with the `InitFn` type, which is a function of type `void(Library&)`.

When defining `TORCH_LIBRARY_IMPL_static_init_aten_AutogradPrivateUse1_20`, the first parameter `Library::Kind kind` is `torch::Library::IMPL`, and the second parameter is:

```cpp
c10::guts::if_constexpr<c10::impl::dispatch_key_allowlist_check(c10::DispatchKey::AutogradPrivateUse1)>(                                       
          []() {return &TORCH_LIBRARY_IMPL_init_aten_AutogradPrivateUse1_20;},           
          []() { return [](torch::Library&) -> void {}; }
          ),                                                      // Parameter 2, InitFn* type
```

Let's first look at the template parameter `c10::impl::dispatch_key_allowlist_check(c10::DispatchKey::AutogradPrivateUse1)`, which is defined as:

```cpp
constexpr bool dispatch_key_allowlist_check(DispatchKey /*k*/) {
#ifdef C10_MOBILE
  return true;
  // Disabled for now: to be enabled later!
  // return k == DispatchKey::CPU || k == DispatchKey::Vulkan || k == DispatchKey::QuantizedCPU || k == DispatchKey::BackendSelect || k == DispatchKey::CatchAll;
#else
  return true;
#endif
}
```

As we can see, it currently unconditionally returns `true`. Therefore, the second parameter becomes:

```cpp
c10::guts::if_constexpr<true>(                                       
          []() {return &TORCH_LIBRARY_IMPL_init_aten_AutogradPrivateUse1_20;},           
          []() { return [](torch::Library&) -> void {}; }
          ),                                                      // Parameter 2, InitFn* type
```

The definition of `if_constexpr` is as follows:

```cpp
template <bool Condition, class ThenCallback, class ElseCallback>
decltype(auto) if_constexpr(
    ThenCallback&& thenCallback,
    ElseCallback&& elseCallback) {
    // ...
}
```

Here's a simplified version of the code:

```cpp
void TORCH_LIBRARY_IMPL_init_aten_AutogradPrivateUse1_20(torch::Library & m) {
    m.impl(<myadd_schema>, &myadd_autograd);
}
```

The complete code before simplification was:

```cpp
TORCH_LIBRARY_IMPL(aten, AutogradPrivateUse1, m) {
  m.impl(<myadd_schema>, &myadd_autograd);
}
```

After macro expansion and simplification, it becomes:

```cpp
static void TORCH_LIBRARY_IMPL_init_aten_AutogradPrivateUse1_20(torch::Library & m);

static const torch::detail::TorchLibraryInit TORCH_LIBRARY_IMPL_static_init_aten_AutogradPrivateUse1_20(          
  torch::Library::IMPL,
  &TORCH_LIBRARY_IMPL_init_aten_AutogradPrivateUse1_20,
  "aten",
  c10::make_optional(c10::DispatchKey::AutogradPrivateUse1),
  __FILE__,
  __LINE__
);

void TORCH_LIBRARY_IMPL_init_aten_AutogradPrivateUse1_20(torch::Library & m) {
   m.impl(<myadd_schema>, &myadd_autograd);
}

class TorchLibraryInit final {
 private:
  using InitFn = void(Library&);
  Library lib_;

 public:
  TorchLibraryInit(
      Library::Kind kind,
      InitFn* fn,
      const char* ns,
      c10::optional<c10::DispatchKey> k,
      const char* file,
      uint32_t line)
      : lib_(kind, ns, k, file, line) {
    fn(lib_);
  }
};
```

Summary:

**① The first part declares a static function called `TORCH_LIBRARY_IMPL_init_aten_AutogradPrivateUse1_20`.**

**② The second part declares a static constant of type `torch::detail::TorchLibraryInit` named `TORCH_LIBRARY_IMPL_static_init_aten_AutogradPrivateUse1_20`. It has a member variable of type `Library` that is initialized using the parameters passed and the static function declared in the first part.**

**③ The third part implements the function declared in the first part. Note that this function registers operators by invoking the `impl` member function of the `torch::Library` parameter. The actual arguments passed to this function are the private member variables of the static constant declared in the second part, where the name of the static constant depends on the namespace, device (cpu or cuda or XXX), and UID.**

The constructor of `TORCH_LIBRARY_IMPL_static_init_aten_AutogradPrivateUse1_20` initializes its private member variable `lib_` using `TORCH_LIBRARY_IMPL_init_aten_AutogradPrivateUse1_20`. This initialization is accomplished by invoking the `impl` function of its `torch::Library` class's private member variable `lib_`.

Now let's explain the `impl` method of the `torch::Library` class, as defined below:

```cpp
/// Register an implementation for an operator.  You may register multiple
/// implementations for a single operator at different dispatch keys
/// (see torch::dispatch()).  Implementations must have a corresponding
/// declaration (from def()), otherwise they are invalid.  If you plan
/// to register multiple implementations, DO NOT provide a function
/// implementation when you def() the operator.
///
/// \param name The name of the operator to implement.  Do NOT provide
///   schema here.
/// \param raw_f The C++ function that implements this operator.  Any
///   valid constructor of torch::CppFunction is accepted here;
///   typically you provide a function pointer or lambda.
///
/// ```
/// // Example:
/// TORCH_LIBRARY_IMPL(myops, CUDA, m) {
///   m.impl("add", add_cuda);
/// }
/// ```
template <typename Name, typename Func>
Library& impl(Name name, Func&& raw_f, _RegisterOrVerify rv = _RegisterOrVerify::REGISTER) & {
  // TODO: need to raise an error when you impl a function that has a
  // catch all def
#if defined C10_MOBILE
  CppFunction f(std::forward<Func>(raw_f), NoInferSchemaTag());
#else
  CppFunction f(std::forward<Func>(raw_f));
#endif
  return _impl(name, std::move(f), rv);
}
```

Clearly, this is a function that utilizes universal reference and perfect forwarding (something I saw for the first time except in interviews). Internally, it first creates an object of type `CppFunction`. Since the registration method is:

```cpp
TORCH_LIBRARY_IMPL(aten, AutogradPrivateUse1, m) {
  m.impl(<myadd_schema>, &myadd_autograd);
}
```

The initial constructor of `CppFunction` that is called is as follows:

```cpp
template <typename Func>
explicit CppFunction(
    Func* f,
    std::enable_if_t<
        c10::guts::is_function_type<Func>::value,
        std::nullptr_t> = nullptr)
    : func_(c10::KernelFunction::makeFromUnboxedRuntimeFunction(f)),
      cpp_signature_(c10::impl::CppSignature::make<Func>()),
      schema_(
          c10::detail::inferFunctionSchemaFromFunctor<std::decay_t<Func>>()),
      debug_() {}
```

As can be seen, it initializes the private member variables `func_`, `cpp_signature_`, and `schema_`, which correspond to function, signature, and schema, respectively. The definition of the `_impl` function is as follows:

It can be seen that in the initialization list, the private member variables `func_`, `cpp_signature_`, and `schema_` are initialized. As the names suggest, they correspond to the function, signature, and schema, respectively. The definition of the `_impl` function is as follows:

```cpp
Library& Library::_impl(const char* name_str, CppFunction&& f, _RegisterOrVerify rv) & {
  at::OperatorName name = _parseNameForLib(name_str);
  // See Note [Redundancy in registration code is OK]
  TORCH_CHECK(!(f.dispatch_key_.has_value() &&
                dispatch_key_.has_value() &&
                *f.dispatch_key_ != *dispatch_key_),
    IMPL_PRELUDE,
    "Explicitly provided dispatch key (", *f.dispatch_key_, ") is inconsistent "
    "with the dispatch key of the enclosing ", toString(kind_), " block (", *dispatch_key_, ").  "
    "Please declare a separate ", toString(kind_), " block for this dispatch key and "
    "move your impl() there.  "
    ERROR_CONTEXT
  );
  auto dispatch_key = f.dispatch_key_.has_value() ? f.dispatch_key_ : dispatch_key_;
  switch (rv) {
    case _RegisterOrVerify::REGISTER:
      registrars_.emplace_back(
        c10::Dispatcher::singleton().registerImpl(
          std::move(name),
          dispatch_key,
          std::move(f.func_),
          std::move(f.cpp_signature_),
          std::move(f.schema_),
          debugString(std::move(f.debug_), file_, line_)
        )
      );
      break;
    case _RegisterOrVerify::VERIFY:
      c10::Dispatcher::singleton().waitForImpl(name, dispatch_key);
      break;
  }
  return *this;
}
```

It can be seen that the operator is registered with the Dispatcher by calling the `registerImpl` member function of the global singleton of type `Dispatcher`.

**In summary:**
**Every time an operator is registered using the `TORCH_LIBRARY_IMPL` macro, a globally unique static variable of type `TorchLibraryInit` is generated, and the initial constructor of this static variable calls the globally unique function, thereby registering the operator with the Dispatcher. The Dispatcher in PyTorch is responsible for allocating the corresponding backend operator based on various information about the tensor.**