# Integration Marketplace
A catalog of ~87 standalone, copy-ready playbooks for integrating third-party services and common app patterns.
Each playbook has prerequisites, setup steps, a generic code pattern, env var names (never values) and gotchas.
Categories: auth, database, messaging, AI/LLM, maps, payments, push, analytics, hosting, storage and more.
Browse `catalog/README.md` for the index, or open `catalog/<category>/<integration>.md` directly.
Hand one category folder to an AI coding session and it can implement the integration from scratch.
Every file records an adopter count and implementation variants, never project names or live-system details.
`catalog/SCHEMA.md` defines the playbook format, confidence levels (high/medium/low) and categories.
New playbooks are drafted with the `map-integrations` Claude Code skill and checked by `audit-catalog-leaks`.
Documentation only — no code, no build. Contributions: follow SCHEMA.md and keep it identifier-free.
