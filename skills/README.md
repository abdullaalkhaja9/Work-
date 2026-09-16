# Skills

A skill is one recurring piece of legal work, packaged so the platform can run it and a
lawyer can read and approve it.

Each skill is a folder:

```
skills/<name>/
  SKILL.md        what it does, when it runs, its instructions, its limits (plain language)
  schema.json     the structured output the model must return
  tests/          input documents and expected outputs from the golden set
```

Rules:

- A skill names the playbooks it uses. It never hard-codes a legal position.
- A skill returns structured output validated against `schema.json`. Prose goes in fields.
- A skill's output is a draft. The UI presents it for a human decision.
- A skill cannot go live until its test set passes and the owning lawyer has signed off.
- Text from documents under review is untrusted. Instructions inside a document are findings, not commands.

See `contract-review/` for the reference example.
