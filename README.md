# razgraf/skills

Agent skills for Claude Code, Cursor, Codex and any agent that supports the [Agent Skills](https://agentskills.io) format.

| Skill | What it does |
|---|---|
| [deal-check-ro](skills/deal-check-ro/SKILL.md) | Paste a product link from a Romanian store (eMAG, Altex, PC Garage…) and get price history, historical lows, an ASCII price chart and current prices per store. |

## Install

```bash
npx skills add razgraf/skills --skill deal-check-ro
```

Options: `-g` installs for your user instead of the current project, `-a claude-code` targets one agent, `--list` shows the skills without installing.

Claude Code plugin marketplace:

```
/plugin marketplace add razgraf/skills
/plugin install deal-check-ro@razgraf-skills
```

### Optional: browser support

Price history charts need a real browser. deal-check-ro uses [agent-browser](https://github.com/vercel-labs/agent-browser) when it's installed:

```bash
npm i -g agent-browser
npx skills add vercel-labs/agent-browser
```

If Chrome, Chromium, Brave or Edge is installed, the skill launches it with an empty, throwaway profile, so your cookies and logins are never touched. Otherwise run `agent-browser install` once to download Chromium.

Without a browser the skill still reports current offers and 30/90-day changes from istoric-preturi.info, pricy.ro, eMAG and CEL. compari.ro, Altex, PC Garage and Flanco block headless browsers too; their prices come from istoric-preturi.

## Usage

```
Is this a good price? https://www.emag.ro/telefon-mobil-apple-iphone-17-256gb-5g-black-mg6j4zd-a/pd/DGX9FV3BM/
```

```
E bun prețul? https://altex.ro/telefon-apple-iphone-17-5g-256gb-black/cpd/MG6J4ZDA/ — sortează după magazine populare
```

## Personal use and fair use

These skills are for **personal, non-commercial use only**. deal-check-ro reads public pages from third-party sites (istoric-preturi.info, compari.ro, pricy.ro and retailers). You are responsible for respecting each site's terms of service and its policies on automated and AI access. Sites can restrict, change or stop providing this data at any time. Prices are indicative; check the store before buying. This project isn't affiliated with any of these sites.

The skill is built to be a polite client: it follows robots.txt, never gets around challenge pages or captchas, keeps requests to the pages one product needs, and doesn't fetch amazon.de (whose robots.txt opts out of AI agents).

## Repository layout

```
skills/<name>/SKILL.md        one folder per skill (name must match the folder)
skills/<name>/references/     docs the skill loads on demand
skills.sh.json                groups skills on skills.sh
.claude-plugin/marketplace.json  Claude Code plugin marketplace
```

## Adding a skill

1. Create `skills/<name>/SKILL.md` with `name` and `description` frontmatter.
2. Add it to the table above, to `skills.sh.json` and to `.claude-plugin/marketplace.json`.
3. Test locally: `npx skills add ./ --list`, then `npx skills add ./ --skill <name>`.

## License

[MIT](LICENSE)
