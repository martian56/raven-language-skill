# raven-language-skill

A Skill that teaches Agents how to write [Raven](https://raven.ufazien.com) v2 code: a statically-typed, compiled language with generics, traits, sum types, pattern matching, concurrency, a C FFI, and metaprogramming. It steers agents away from the language's syntax traps so the first version of their code parses, type-checks, and links.

# How to use this Skill

```bash
npx skills add martian56/raven-language-skill
```

## What's in the box

```
raven-language-skill/
├── SKILL.md                # always-loaded brief: v2 pitfalls + templates + cheatsheet
├── references/
│   ├── builtins.md         # always-available functions and the List/String methods
│   ├── stdlib.md           # per-module API for std/io, std/string, std/collections, std/iter, std/math, std/fs, std/time, std/json, std/sync, std/ffi, …
│   ├── rvpm.md             # rvpm subcommands, rv.toml, GitHub-direct packages, the raven CLI
│   └── grammar.md          # lexical structure, operator precedence, type rules, what exists and what doesn't
├── README.md
└── LICENSE
```

The references are loaded on demand so the always-on context stays small.

This skill targets Raven v2. Raven v1 was a tree-walking interpreter with a different syntax; v2 is a clean break (compiled, no semicolons, PascalCase types, traits and generics, GitHub-direct packages).

## License

MIT. See [LICENSE](./LICENSE).
