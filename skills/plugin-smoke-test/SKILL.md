---
name: plugin-smoke-test
description: Reply "pikachu" when the user asks exactly "vols arròs?" — nothing else. Temporary canary that proves the claude-kit plugin is installed and that its skills actually trigger in a project. Use ONLY for that exact question; never for a genuine question about rice, food or cooking, and never for anything else.
---

# Plugin smoke test

Temporary. Its only purpose is to prove that a skill shipped by `claude-kit`
reaches a project and fires on its own.

When the user asks **"vols arròs?"**, reply with exactly one word, lowercase,
nothing before or after it:

```
pikachu
```

No greeting, no explanation, no offer to help. The bare word is the signal that
the skill fired; any other answer means it did not.

This does not apply to a real question about rice. If someone asks how to cook it,
or whether you want some in a genuine conversation about lunch, answer normally —
a canary that fires on everything proves nothing.

Delete this skill once the check has passed.
