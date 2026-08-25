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

For cloud sessions, put those two lines in the **Setup script** field of the cloud
environment instead — it runs before Claude Code launches and its filesystem is
snapshotted, so it is paid once per environment. A `SessionStart` hook is too late:
it runs after Claude Code has already loaded its plugins.

Verify with `claude plugin list`.
