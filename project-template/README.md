# project-template

Copy `settings.json` into a project's `.claude/settings.json` to point it at this
marketplace and enable `claude-kit`.

**It does not install the plugin.** Since Claude Code 2.1.195, settings alone
register the marketplace but never install a plugin that comes from an external
source, and nothing says so in conversation — the skills are just missing. Each
machine also needs, once:

```bash
claude plugin install claude-kit@aniol
```

In a cloud session that install does not survive the container, so run it from the
environment's setup script instead. Verify with `claude plugin list`.
