# Saffron for Claude Code

One plugin gives Claude Code everything it needs to write and maintain
[Saffron](https://saffron-ai.lovable.app) tests well:

- **Language server** for `.saffron` and `.feature` files: Claude sees
  diagnostics after each edit (unknown or duplicate StepSet names, a set
  name missing the keyword, near-duplicate wordings), and gets completion
  with recorded / divergent / unrecorded badges, StepSet and `{env:VAR}`
  go-to-definition, and hover.
- **MCP server** with `search_steps`, `list_step_sets` and
  `project_status`, so Claude reuses your recorded wordings and knows what
  is cached, pending review and tagged.
- **The Saffron agent skill**: reuse the vocabulary, keep `Then` steps
  sacred, secrets as `{env:VAR}`, StepSet syntax, the run → review →
  accept workflow.

## Install

```
/plugin marketplace add s-chathuranga-j/saffron-claude-plugin
/plugin install saffron@saffron
```

The language server and MCP server run `saffron-ai` through `npx`,
pinned to the version this plugin was built from; the first start
downloads it. Recording and healing need Claude credentials in the
project as usual (`npx saffron run`); the plugin itself needs nothing.

## Source

Built from the `claude-plugin/` folder of the Saffron runner. Issues:
https://github.com/s-chathuranga-j/saffron-ai/issues.
