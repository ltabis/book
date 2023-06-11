Traits
======

{{#include ../links.md}}

A number of traits, under the `rhai::` module namespace, provide additional functionalities.

| Trait                    | Description                                                        | Methods                                                                                      |
| ------------------------ | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `CustomType`             | trait to build a [custom type] for use with an [`Engine`]          | `build`                                                                                      |
| `Func`                   | trait for creating Rust closures from script                       | `create_from_ast`, `create_from_script`                                                      |
| `FuncArgs`               | trait for parsing function call arguments                          | `parse`                                                                                      |
| `ModuleResolver`         | trait implemented by [module resolution][module resolver] services | `resolve`, `resolve_ast`, `resolve_raw`                                                      |
| `packages::Package`      | trait implemented by [packages]                                    | `init`, `init_engine`, `register_into_engine`, `register_into_engine_as`, `as_shared_module` |
| `plugin::PluginFunction` | trait implemented by [plugin] functions                            | `call`, `is_method_call`, `has_context`, `is_pure`                                           |
