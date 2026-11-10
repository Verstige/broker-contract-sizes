# Contract Sizes: Make Your P&L Match the Broker

> Stop your P&L disagreeing with your broker. The verification workflow for contract sizes, why every broker differs, and how to audit a table that survived a broker migration.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg) ![Claude Code](https://img.shields.io/badge/Claude_Code-skill-blueviolet.svg) [![Available on Agensi](https://img.shields.io/badge/Agensi-%2412-ff5a5f.svg)](https://agensi.io/skills/contract-sizes-make-your-p-l-match-the-broker)

## What it does

Verify a broker's contract_size per instrument so your computed P&L matches the broker-reported number. Audit the contract-size table that survived a broker migration.

A CFD broker's `contract_size` per instrument is the multiplier
that converts `lots × price_diff` into broker-reported P&L.
Every instrument class (crypto, indices, metals, FX, energy) has
its own convention and **every broker differs**:

## The failure it prevents

The cases that cost time are specific, and they are written down:

- **Don't override a verified value based on convention alone.**
- **Don't commit a contract_size change without a test trade.**
- **Don't keep the old contract_size in a comment as "previous
- **Don't use `*1000` or `*10` multipliers as band-aids.**
- **Don't trust the risk-calculator view or any other doc that

## When to use it

- Closed-trade P&L in the app doesn't match the broker's reported
- Porting a contract-size table from one broker to another
- Adding a new instrument class (crypto, indices, metals, FX,
- Auditing an existing `contract-sizes.ts` file for values that

## What's inside

```
plugins/
└── broker-contract-sizes/
    ├── .claude-plugin/plugin.json
    └── skills/broker-contract-sizes/
        ├── SKILL.md
```

## Install

Add this repo as a Claude Code marketplace, then install the skill:

```bash
claude plugin marketplace add Verstige/broker-contract-sizes
claude plugin install broker-contract-sizes@broker-contract-sizes
```

Or copy the skill straight into your skills directory:

```bash
git clone https://github.com/Verstige/broker-contract-sizes.git
cp -R broker-contract-sizes/plugins/broker-contract-sizes/skills/broker-contract-sizes ~/.claude/skills/
```

## Usage

The skill activates on the work it describes. Ask for the outcome, and it
supplies the method:

```
> "make closed-trade P&L match the broker's own report"
```

## License

MIT. See [LICENSE](LICENSE).

---

**Also available as a paid skill on Agensi** — $12. Install it with one command, no clone required:

[agensi.io/skills/contract-sizes-make-your-p-l-match-the-broker](https://agensi.io/skills/contract-sizes-make-your-p-l-match-the-broker)

Part of the [Verstige](https://github.com/Verstige) skills portfolio. More at [github.com/Verstige](https://github.com/Verstige).
