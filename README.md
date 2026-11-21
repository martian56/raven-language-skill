# raven-language-skill

A Skill that teaches agents how to write [Raven](https://raven.ufazien.com) 3 code. Raven reads like Python with braces and compiles to native code, with classes, traits, generics, sum types, pattern matching, concurrency, a C FFI, and metaprogramming. The skill steers agents past the syntax traps so the first version of their code compiles.

# How to use this Skill

```bash
npx skills add martian56/raven-language-skill
```

## What's in the box

```
raven-language-skill/
├── SKILL.md                # always-loaded brief: pitfalls, templates, workflow
├── references/
│   ├── builtins.md         # builtins and the str, list, dict, set, Option, Result methods
│   ├── stdlib.md           # per-module API for the std.* modules
│   ├── rvpm.md             # rvpm and raven commands, rv.toml, imports and packages
│   └── grammar.md          # lexical rules, precedence, declarations, patterns, types
├── README.md
└── LICENSE
```

The references load on demand so the always-on context stays small.

This skill targets Raven 3. Raven 2 used an older syntax (`struct`, `Int`, `List<T>`, `&&`, `import std/x { }`) that Raven 3 no longer compiles.

## License

MIT. See [LICENSE](./LICENSE).
