# BytePlus Ark CLI

BytePlus Ark CLI provides command-line access and Agent Skills for BytePlus Ark. This repository contains the public English Skills distributed with the CLI and the documentation required to use them from supported AI coding agents.

## Install

```shell
npm install -g @byteplus/ark-cli@latest
```

After installing the CLI, authenticate and install the bundled Skills into the supported local agents:

```shell
arkcli auth login
arkcli auth status
arkcli +connect
```

BytePlus CLI state is stored separately under `~/.arkcli-bp`.

## Skills

The [`skills/`](./skills) directory contains one capability-oriented Skill per Ark CLI domain. Each Skill uses the standard `SKILL.md` and `references/` layout and is written in English for BytePlus users.

The public Skill tree is derived from the Ark CLI source repository at a fixed commit. Product support is audited independently for BytePlus; the presence of a Skill documents the command surface and does not override the support status recorded by the product capability audit.

## Community

Scan the QR code to join the Ark CLI Lark user group for installation help, troubleshooting, bug reports, and usage discussions.

Do not share API keys, access keys, tokens, or other credentials in the group. For reproducible bugs, please also open a GitHub issue so the fix can be tracked.

<img src="./assets/lark-user-group.png" width="360" alt="Ark CLI Lark user group QR code" />

## Security

Do not disclose security issues through a public GitHub issue or the user group. Report them through the official BytePlus support channel.

## Links

- npm: <https://www.npmjs.com/package/@byteplus/ark-cli>
- GitHub Releases: <https://github.com/byteplus-sdk/ark-cli/releases>
- Bundled Skills: <https://github.com/byteplus-sdk/ark-cli/tree/main/skills>

## License

The public Skills and documentation are licensed under the [Apache License 2.0](./LICENSE).
