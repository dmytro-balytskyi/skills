# Agent Skills Collection

Open-source agent skills for AI coding assistants. Built by a freelance developer & tech enthusiast, shared with the community.

Vue.js • GraphQL • Fastify • TypeScript

## 📦 Available Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [vue-apollo](./skills/vue-apollo/) | Vue 3 + GraphQL best practices with @vue/apollo-composable & Apollo Client v3 | `npx skills add dmytro-balytskyi/skills --skill vue-apollo` |

> **More skills coming soon!** Contributions welcome.

## 🚀 Installation

Install any skill from this repository:

```bash
# Install a specific skill globally
npx skills add https://github.com/dmytro-balytskyi/skills --skill vue-apollo -g -y
```

## 🏗️ Structure

This is a monorepo containing multiple agent skills. Each skill lives in its own directory:

```
skills/
├── vue-apollo/                  # Vue 3 + GraphQL skill
│   ├── SKILL.md                 # Core instructions
│   └── references/              # Detailed guides
│       ├── setup.md             # Apollo Client configuration
│       ├── queries.md           # useQuery, useLazyQuery patterns
│       ├── mutations.md         # useMutation & cache updates
│       └── subscriptions-pagination.md
├── [your-skill-here]/           # Your next skill goes here
│   ├── SKILL.md
│   └── references/
└── ...
```

## 🎯 What's Inside

Each skill follows best practices for agent skills:

- **Concise** — Under 500 lines in SKILL.md (progressive disclosure)
- **Type-safe** — Full TypeScript support with real-world examples
- **Well-documented** — Clear anti-patterns, integration graphs, and quick references
- **Battle-tested** — Based on actual freelance project experience

## 🤝 Contributing

Contributions are welcome! Whether you want to:

- Add a new skill
- Improve an existing one
- Fix typos or broken examples

Just open a Pull Request. I review all PRs personally as a solo maintainer.

## 📜 License

MIT — Free to use, fork, and improve.

---

Built with ❤️ by [Dmytro Balytskyi](https://github.com/dmytro-balytskyi) — freelance developer & open-source enthusiast.
