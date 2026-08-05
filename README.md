# CANOA Skills

Open skills for working with **CANOA — Corpus Apertum Nummorum Orbis Antiqui**
(https://canoanumis.org), an open corpus of ancient world coinage: Greece, Rome,
Persia, Carthage, Byzantium and more.

The corpus aggregates ~128,500 coin types, 540,000+ museum specimens, 9,833
hoards, 17,074 dies and 2,600+ mints from the numismatics.org ecosystem and
museum collections. Everything is exposed through a free, key-less JSON API —
see the skills for how to use it.

## Contents

| Skill | What it does |
|---|---|
| [`skills/canoa-api/`](skills/canoa-api/SKILL.md) | How to use the CANOA API for numismatics, history, education — endpoints, filters, recipes for kids, hobbyists and researchers, citation and licensing rules |

## Installation

With [Hermes Agent](https://hermes-agent.nousresearch.com) (or any Agent Skills
client), install from the public catalog:

```bash
hermes skills search canoa-api
hermes skills install skills-sh/canoa-dev/canoa-skills/skills/canoa-api
```

Or directly from this repository:

```bash
hermes skills install canoa-dev/canoa-skills/skills/canoa-api
```

The skill needs no API key and no setup — just read it and follow the examples
(`curl`, `python3`).

## Quick tour

```bash
# Coins of Nero struck in Rome:
curl "https://canoanumis.org/api/coins?authority=http://nomisma.org/id/nero&mint=36&per_page=100"

# Coins of Rome 54–68 AD:
curl "https://canoanumis.org/api/coins?date_from=54&date_to=68&mint=36"

# Coin card:
curl "https://canoanumis.org/api/coins/ric-i-second-edition-nero-63/"

# Mints as GeoJSON:
curl "https://canoanumis.org/api/mints.geojson?has_coins=1"
```

Full endpoint reference and recipes are inside
[`skills/canoa-api/SKILL.md`](skills/canoa-api/SKILL.md).

## Data and licensing

- Coin type data — **Open Database License (ODbL) 1.0**, source: numismatics.org projects (OCRE, CRRO, PELLA, …)
- Specimen images — © the holding museums; many collections allow non-commercial use only with attribution
- Per-collection licenses: https://canoanumis.org/LICENSES.md
- This skill itself — **CC BY 4.0** (see [LICENSE](LICENSE))

When publishing results, credit: *Data: CANOA (canoanumis.org), ODbL; image: © &lt;museum&gt;*.

## Contributing

Skills are plain folders with a `SKILL.md` (Agent Skills format). To add a skill:

1. Create `skills/<name>/SKILL.md` with `name:` and `description:` frontmatter
2. Test it: `hermes chat --toolsets skills -q "Use the <name> skill to ..."`
3. Open a pull request

New skills about ancient numismatics, history, museum data and education are
welcome. Before publishing, run `hermes skills publish` — the built-in safety
scan must pass.

## Links

- Corpus: https://canoanumis.org
- API docs: https://canoanumis.org/llms-full.txt
- Licenses: https://canoanumis.org/LICENSES.md
- Skills Hub: https://hermes-agent.nousresearch.com/docs/skills/
- Agent Skills spec: https://agentskills.io

Contact: info@canoanumis.org (suggestions), abuse@canoanumis.org (complaints, copyright).
