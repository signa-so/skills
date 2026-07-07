# Signa Agent Skills

[Agent skills](https://www.skills.sh) for building with the [Signa API](https://docs.signa.so) — global trademark search, clearance screening, and monitoring.

## Install

```bash
npx skills add signa-so/skills
```

Works with Claude Code, Cursor, Codex, Copilot, Windsurf, Gemini CLI, and any agent that supports the [skills format](https://github.com/vercel-labs/skills).

## Skills

### `signa`

Teaches your coding agent the Signa API: authentication, request conventions, the response envelope, search strategies and filters, cursor pagination, rate limits, idempotency, error handling, the monitoring stack (watches → alerts → webhooks), and the TypeScript SDK (`@signa-so/sdk`) — plus how to pull always-current reference from the live docs.

With this skill installed, your agent writes correct Signa integrations on the first try instead of guessing parameter names.

## What is Signa?

Signa is an API-first platform for global trademark intelligence: search trademarks across the world's trademark offices — USPTO, EUIPO, WIPO, and more; screen brand names for clearance risk; monitor new filings with watches, alerts, and webhooks.

- [Documentation](https://docs.signa.so)
- [Get an API key](https://app.signa.so)
- [TypeScript SDK](https://www.npmjs.com/package/@signa-so/sdk)

## License

MIT
