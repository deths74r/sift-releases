# Custom Workflows

Automate repetitive tasks by describing what you want to Claude.

---

## The Process

1. **Tell Claude what you're tired of doing manually**
2. **Talk through what you want to happen**
3. **Claude edits CLAUDE.md and stores the pattern in memory**
4. **Use it**

You don't write the workflow. You describe it. Claude implements it.

---

## Example: How We Built the Branch Workflow

**The conversation went something like this:**

> "I keep forgetting which branch goes with which piece of work. And I want branch names that link back to the plan somehow."

Claude proposed embedding the memory ID in the branch name. We talked about the format - settled on `{type}/{hash}-{hint}` where the hash comes from the plan's memory ID.

> "What phrases should trigger this?"

We listed natural ways to start work: "let's work on X", "new feature for X", "fix the bug in X".

Claude then:
- Edited `CLAUDE.md` to add the workflow definition
- Stored the pattern in memory so it persists
- Started using it immediately

**The result:**

```markdown
## Branch Workflow

**Trigger signals** (when on main branch and work is stable):
- "Let's work on X"
- "Start working on X"
- "New feature for X"
- "Fix the X bug"
- "Create a branch for X"

**When triggered:**
1. Create memory plan first → get `mem-{hash}...`
2. Use first 6 chars of memory ID as branch hash
3. Ask for type (feat/fix/chore/refactor/docs/test)
4. Ask for hint (1-3 word kebab-case)
5. Branch name = `{type}/{hash}-{hint}`
6. Create git branch: `git checkout -b {branch}`
7. Update plan title to match branch name
```

Now saying "let's work on dark mode" triggers the whole sequence automatically.

---

## Starting Your Own

Just tell Claude:

- "I keep having to do X manually, can we automate it?"
- "When I say Y, I want you to do Z"
- "Set up a workflow for [task]"

Claude will ask clarifying questions, propose a design, edit your CLAUDE.md, and store it in memory. Then it just works.

---

## Where Workflows Live

| Location | Scope |
|----------|-------|
| `CLAUDE.md` | The trigger definitions |
| Memory | The pattern, persisted across sessions |

Claude manages both. You just use them.
