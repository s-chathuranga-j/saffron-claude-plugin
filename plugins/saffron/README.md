# Saffron for Claude Code

One plugin gives Claude Code everything it needs to write and maintain
[Saffron](https://saffron-ai.io) tests well:

- **Language server** for `.saffron` and `.feature` files: Claude sees
  diagnostics after each edit (unknown or duplicate StepSet names, a set
  name missing the keyword, near-duplicate wordings), and gets completion
  with recorded / divergent / unrecorded badges, StepSet and `{env:VAR}`
  go-to-definition, and hover.
- **MCP server** with `search_steps`, `list_step_sets` and
  `project_status`, so Claude reuses your recorded wordings and knows what
  is cached, pending review and tagged.
- **The Saffron agent skill**: reuse the vocabulary, keep `Then` steps
  sacred, secrets as `{env:VAR}` and shared values as `{data:...}`,
  StepSets, several people in one scenario (`As "buyer"`), the
  run → review → accept workflow, and reviewing a heal made in CI.

## Install

```
/plugin marketplace add s-chathuranga-j/saffron-claude-plugin
/plugin install saffron@saffron
```

The language server and MCP server run `saffron-ai` through `npx`,
pinned to the version this plugin was built from; the first start
downloads it. Recording and healing need Claude credentials in the
project as usual (`npx saffron run`); the plugin itself needs nothing.

### Codex

```
codex plugin marketplace add s-chathuranga-j/saffron-claude-plugin
codex plugin add saffron@saffron
```

Codex gets the MCP server and the skill; its plugins have no language
server slot.

## License

Free to install and use, including commercially, under the
[Saffron Free Use License v1.0](LICENSE). The tests, caches, reports and
other files Saffron produces from your work are yours. You may not
modify or redistribute Saffron itself. AI calls run on your own
credentials, so any charges are on your account.

## Source

Built from the `claude-plugin/` folder of the Saffron runner. Issues:
https://github.com/s-chathuranga-j/saffron-ai/issues.
