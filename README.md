# raven-language-skill

A Skill that teaches Agents how to write [Raven](https://raven.ufazien.com) code without falling into the language's syntax traps on the first try.

## What's in the box

```
raven-language-skill/
├── SKILL.md                # the always-loaded brief: pitfalls + templates + cheatsheet
├── references/
│   ├── builtins.md         # full signatures for built-in functions (print, file I/O, time, network, …)
│   ├── stdlib.md           # per-module API for math, str, collections, filesystem, json, …
│   ├── rvpm.md             # rvpm subcommands, rv.toml format, raven CLI flags, module resolution
│   └── grammar.md          # lexical structure, operator precedence, type rules, things that don't exist
├── README.md
└── LICENSE
```

The references are loaded on demand so the always-on context stays small.

## License

MIT. See [LICENSE](./LICENSE).
