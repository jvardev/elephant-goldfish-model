I would adapt EGM as a global OpenCode V2 skill package, rather than trying to install it as a Claude or Codex plugin.

After inspecting the current EGM repository and OpenCode V2 documentation, I think this is quite feasible. You do not need to rewrite the framework. You need a relatively small OpenCode compatibility layer.

There are three things to adapt:

1. Skill discovery and resource resolution: OpenCode must find EGM's skills, personas and document standards.
2. Agent delegation: EGM uses named personas and fresh-context subagents. OpenCode uses its own `subagent` tool.
3. Invocation: Replace the Claude/Codex slash-command conventions with OpenCode skill invocation.

One distinction worth mentioning: EGM's harness documentation lists Antigravity, Claude, Codex, ChatGPT and `omp`, but not OpenCode. Its README does encourage installation with any AI agent. So I'd consider this a custom OpenCode integration, not an officially tested one.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+1



## 1. Install EGM as a global package

Clone the complete repository, not just its `skills` directory:

```
git clone https://github.com/google/elephant-goldfish-model.git \
  ~/.config/opencode/egm
```

Keep the original directory structure:

```
~/.config/opencode/egm/
├── agents/
├── docs/
├── resources/
│   └── document_standards/
├── skills/
│   ├── egm-brainstorm/
│   │   └── SKILL.md
│   ├── egm-draft/
│   │   └── SKILL.md
│   ├── egm-review/
│   │   └── SKILL.md
│   ├── egm-review-loop/
│   │   └── SKILL.md
│   ├── egm-resolve/
│   │   └── SKILL.md
│   ├── egm-bootstrap/
│   │   └── SKILL.md
│   └── egm-drift-report/
│       └── SKILL.md
└── ...
```

EGM specifically requires the shared `agents/` and `resources/document_standards/` directories. Installing only the skills breaks some workflows.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+1



Now configure OpenCode globally:

`~/.config/opencode/opencode.json`

```
{
  "$schema": "https://opencode.ai/config.json",
  "skills": [
    "~/.config/opencode/egm/skills"
  ]
}
```

OpenCode V2 supports additional skill directories through the `skills` configuration property. The path is resolved from the user's home directory when prefixed with `~/`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://opencode.ai\&sz=32)

OpenCode



## 2. Solve EGM's bundled-file resolution

This is particularly important for your original question.

EGM has two kinds of files:

| Files                               | Resolve relative to        |
| ----------------------------------- | -------------------------- |
| `agents/zoom-out.md`                | EGM repository root        |
| `resources/document_standards/*.md` | EGM repository root        |
| `docs/harnesses.md`                 | EGM repository root        |
| `references/*.md`                   | Individual skill directory |
| `docs/designs/*.md`                 | Active project root        |
| Generated reports and drafts        | Active project root        |

OpenCode V2 has a useful feature here: when it loads a directory-based skill, it provides the model with the skill's base directory and a sample of supporting files.

Additionally, OpenCode documents that paths inside a skill are relative to the directory containing its `SKILL.md`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://opencode.ai\&sz=32)

OpenCode+1



For example:

```
~/.config/opencode/egm/
├── agents/
│   ├── zoom-out.md
│   ├── comprehension.md
│   ├── design.md
│   └── ...
├── resources/
│   └── document_standards/
├── docs/
└── skills/
    └── egm-review/
        └── SKILL.md
```

When OpenCode loads `egm-review`, its skill base directory is:

```
~/.config/opencode/egm/skills/egm-review/
```

Consequently, the EGM repository root is two directories above that base directory:

```
../../
```

You can therefore resolve the files as:

```
../../agents/zoom-out.md
../../agents/comprehension.md
../../resources/document_standards/feature-spec-standard.md
../../docs/harnesses.md
```

These paths are resolved against the EGM skill base directory, not against the active project.

Meanwhile:

```
docs/designs/my-feature.md
```

remains relative to the active project directory.

### Create an OpenCode integration document

I'd add this file to your EGM checkout:

`~/.config/opencode/egm/docs/opencode.md`

Its purpose is to define the path-resolution and delegation conventions for OpenCode.

For example:

```
# EGM - OpenCode V2 Integration

## Path Resolution

OpenCode loads EGM skills from:

<EGM_ROOT>/skills/<skill-name>/SKILL.md

The active skill's base directory is provided by OpenCode
when loading the skill.

Resolve EGM_ROOT as two parent directories above the
active skill's base directory.

Use EGM_ROOT for all framework resources:

- Personas: EGM_ROOT/agents/
- Document standards: EGM_ROOT/resources/document_standards/
- Framework documentation: EGM_ROOT/docs/

Resolve references/ relative to the active SKILL.md directory.

Resolve all user project files and generated artifacts
relative to the active OpenCode working directory.

NEVER resolve EGM framework resources against the project root.

NEVER substitute a similarly named file from the user's
project for a bundled EGM persona or standard.

## Subagent Delegation

Use OpenCode V2's native `subagent` tool.

Use the configured `egm-goldfish` subagent for all EGM
reviewer personas.

The parent agent must explicitly provide:

1. The persona file to read.
2. The target design document.
3. The applicable document standard.
4. The additional GoldfishContext, if provided.
5. The complete review instructions.

The Goldfish subagent must read the persona file first.

Use direct (flat) dispatch from the primary agent.
Do not rely on a synthesizer spawning nested subagents.

Each reviewer must work in a fresh child session.
Do not pass the earlier brainstorming conversation
or the Elephant's conclusions to Goldfish reviewers.

Goldfish reviewers must not modify project files.
Return findings to the primary agent for consolidation.

## Artifact Directory

Unless the user explicitly specifies another location,
write all project artifacts relative to the active
OpenCode working directory.

Follow any EGM artifact directory specified by the user
or in the project's AGENTS.md.
```

Then add a short OpenCode-specific instruction near the beginning of each EGM `SKILL.md`:

```
When running in OpenCode V2, first read the EGM
OpenCode integration guide at:

../../docs/opencode.md

Resolve this path against the skill's base directory
provided by OpenCode.

Follow its path-resolution and subagent delegation
conventions instead of the Claude or Codex conventions.
```

This is the small adaptation that prevents the global-versus-project path problem you described.

## 3. Adapt the personas to OpenCode agents

This is the second important part.

EGM's `agents/` directory contains persona prompts. However, OpenCode doesn't automatically register files from that directory as agents.

OpenCode V2 discovers Markdown agents from:

```
~/.config/opencode/agents/
.opencode/agents/
```

And a Markdown agent must have `mode: subagent` to be callable through the native subagent tool.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://opencode.ai\&sz=32)

OpenCode+1



There are two ways to integrate EGM's personas.

Option A: One generic EGM subagent (my recommendation)

This follows the fallback mechanism EGM already documents for harnesses without named personas: launch a general-purpose subagent and ask it to read the appropriate persona file.

Create:

`~/.config/opencode/agents/egm-goldfish.md`

```
---
description: Executes an EGM Goldfish reviewer persona in an isolated, read-only context.
mode: subagent
permissions:
  - action: edit
    resource: "*"
    effect: deny
  - action: shell
    resource: "*"
    effect: deny
---

You are an EGM Goldfish reviewer.

The parent agent will provide a specific EGM persona file.

Your responsibilities:

1. Read the supplied persona file.
2. Adopt its review perspective and instructions.
3. Read the supplied target document.
4. Read the supplied document standard, if any.
5. Inspect the supplied GoldfishContext.
6. Perform the review independently.

You must not rely on the parent's brainstorming conversation
or undocumented conclusions.

You must not modify project files.

Return your findings with:
- Severity
- Affected section and line numbers
- Discrepancy
- Explanation and rationale

Do not invent findings.
```

Now, instead of asking OpenCode to launch `egm-zoom-out`, `egm-design`, or `egm-structural`, the adapted EGM skill instructs it to launch:

```
subagent:
  agent: egm-goldfish
```

And the prompt specifies the persona file:

```
Read the EGM persona:
<EGM_ROOT>/agents/zoom-out.md

Review:
<PROJECT_ROOT>/docs/designs/my-feature.md

Use the applicable standard:
<EGM_ROOT>/resources/document_standards/feature-spec-standard.md
```

The same generic agent can perform every EGM reviewer role without duplicating the original persona prompts.

Option B: Create individual named OpenCode agents

You could also create:

```
~/.config/opencode/agents/
├── egm-zoom-out.md
├── egm-design.md
├── egm-structural.md
├── egm-style.md
├── egm-comprehension.md
└── ...
```

Each would be an OpenCode adapter referencing the corresponding persona under your global EGM checkout.

This is more convenient for agent selection but requires maintaining additional adapter files.

I would start with Option A.

## 4. Configure subagent permissions

Add this to your global `opencode.json`:

```
{
  "$schema": "https://opencode.ai/config.json",
  "skills": [
    "~/.config/opencode/egm/skills"
  ],
  "permissions": [
    {
      "action": "subagent",
      "resource": "egm-goldfish",
      "effect": "allow"
    }
  ]
}
```

The important detail is that OpenCode V2 has a native `subagent` tool that starts a child session using a configured subagent.

By default, subagent nesting depth is one. This means the primary Elephant can launch Goldfish reviewers, but those Goldfish cannot themselves launch further subagents.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://opencode.ai\&sz=32)

OpenCode+1



Fortunately, EGM's `egm-review-loop` already documents two delegation strategies:

- Hierarchical dispatch through a synthesizer.
- Direct dispatch, where the primary agent launches all the reviewers itself.

For OpenCode, use direct/flat dispatch. It avoids needing nested subagents and preserves the independent reviewer contexts.

This is an important adaptation: don't simply replace Claude's `Agent` tool with OpenCode's `subagent` tool without changing the orchestration strategy.

## 5. How to invoke EGM in OpenCode

OpenCode V2 skills support explicit invocation using `@skill-id`.

For example:

```
@egm-brainstorm Add webhook delivery to this project.
```

```
@egm-draft
```

```
@egm-review docs/designs/webhook-delivery.md
```

The EGM skills use `disable-model-invocation: true` in their frontmatter. OpenCode V2 supports this: it disables automatic invocation but still allows explicit loading by ID.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://dev.opencode.ai\&sz=32)

OpenCode



If you prefer the original slash-command experience, create aliases under:

```
~/.config/opencode/commands/
```

For example, `egm-brainstorm.md`:

```
---
description: Start EGM brainstorming
---

Use the @egm-brainstorm skill.

User request:
$ARGUMENTS
```

OpenCode V2 supports global Markdown commands and argument substitution.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://opencode.ai\&sz=32)

OpenCode



## 6. What the final architecture looks like

Global OpenCode configuration

\~/.config/opencode/

Reusable framework installation

`opencode.json`

`agents/egm-goldfish.md`

`commands/egm-brainstorm.md`

`egm/agents/`

`egm/resources/`

`egm/docs/`

`egm/skills/`

Active OpenCode project

/workspace/my-project/

Project-specific context and generated artifacts

`AGENTS.md`

`README.md`

`docs/designs/`

`src/`

`PROJECT_CONTEXT.md`

Elephant (primary agent)

Holds the project context, conducts brainstorming and drafts specifications.

Native OpenCode subagent calls

Goldfish reviewers (fresh child sessions)

Zoom-out

Design

Structural

The primary agent consolidates findings and manages document changes.

## 7. My recommended implementation order

I would do this in three passes:

| Pass | Work                                        | Expected result                                          |
| ---- | ------------------------------------------- | -------------------------------------------------------- |
| 1    | Global installation and skill discovery     | All seven EGM skills available                           |
| 2    | OpenCode path-resolution adapter            | Skills correctly read global personas and standards      |
| 3    | Generic Goldfish subagent and flat dispatch | Brainstorm and review workflows work with fresh contexts |

Start by testing `egm-draft` and `egm-brainstorm`, then test `egm-review` with a small design document. Verify that its child sessions cannot see the earlier brainstorming conversation and that the reviewers actually read the correct global persona files.

One final clarification about my previous answer: The `instructions` configuration I suggested earlier is useful for automatically loading project documentation, but it isn't the solution to EGM's integration problem. You need to preserve the distinction between the global framework resources and project-relative artifacts, and adapt the delegation mechanism.

### References

- [OpenCode V2 — Skills](https://opencode.ai/v2/docs/skills?utm_source=chatgpt.com)
- [OpenCode V2 — Agents](https://opencode.ai/v2/docs/agents?utm_source=chatgpt.com)
- [OpenCode V2 — Tools and Subagents](https://opencode.ai/v2/docs/tools?utm_source=chatgpt.com)
- [EGM — Codex integration](https://github.com/google/elephant-goldfish-model/blob/main/docs/codex.md?utm_source=chatgpt.com) (especially its resource path-resolution conventions)
- [EGM — Harness architecture](https://github.com/google/elephant-goldfish-model/blob/main/docs/harnesses.md?utm_source=chatgpt.com)

The important design choice is to keep EGM's original skills, personas and standards intact, and implement OpenCode-specific behavior in a thin adapter. That should also make future EGM updates much easier to incorporate.

---

No. They are enough to understand the architecture, but not enough for a reliable, copy-and-paste installation. There are still some important implementation details to settle.

Having checked the actual EGM skills and OpenCode V2 documentation, I would refine the solution as follows.

## What is missing?

There are four important pieces.

1\. Automatic loading of the OpenCode adapter. I suggested creating `docs/opencode.md`, but didn't provide a mechanism that guarantees the EGM skills follow it.

2\. Correct persona resolution. EGM has specific persona filenames. For example, `egm-design` maps to `agents/design_critic.md`, not `agents/design.md`. The integration must inspect the actual files rather than guess filenames.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub



3\. A proper delegation implementation. OpenCode V2 has a native `subagent` tool, but its default nesting depth is one. Consequently, EGM's synthesizer cannot simply launch another layer of Goldfish reviewers. We need direct dispatch from the primary agent.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://dev.opencode.ai\&sz=32)

OpenCode+1



4\. Fresh-context verification. We need to verify that each Goldfish gets an independent child session, without the Elephant's brainstorming conversation being passed to it. OpenCode does not document Codex's `fork_context` or `fork_turns` parameters.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



## What I would actually implement

I would use this structure:

```
~/.config/opencode/
├── opencode.json
├── agents/
│   └── egm-goldfish.md
└── egm/
    ├── agents/                  # Original EGM personas
    ├── docs/
    │   └── opencode.md          # OpenCode compatibility layer
    ├── resources/
    │   └── document_standards/
    └── skills/                  # Original EGM skills
        ├── egm-brainstorm/
        │   └── SKILL.md
        ├── egm-draft/
        │   └── SKILL.md
        ├── egm-review/
        │   └── SKILL.md
        └── ...
```

### 1. Global OpenCode configuration

`~/.config/opencode/opencode.json`

```
{
  "$schema": "https://opencode.ai/config.json",
  "skills": [
    "~/.config/opencode/egm/skills"
  ],
  "instructions": [
    "~/.config/opencode/egm/docs/opencode.md"
  ]
}
```

This is an important improvement over my previous proposal.

OpenCode V2 supports both explicit global skill directories and absolute paths for additional instructions. This means we can keep the original EGM skills untouched and automatically load the compatibility instructions. The compatibility document should specify that its rules apply only to EGM workflows.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://opencode.ai\&sz=32)

OpenCode+1



### 2. One generic Goldfish agent

`~/.config/opencode/agents/egm-goldfish.md`

A reusable OpenCode subagent that accepts a persona file and executes its instructions.

### 3. One compatibility document

`~/.config/opencode/egm/docs/opencode.md`

This handles:

- Finding the EGM root from the loaded skill's base directory.
- Resolving EGM personas and standards against the global installation.
- Resolving design documents and generated artifacts against the current project.
- Translating EGM's reviewer delegation into OpenCode's native subagent calls.
- Enforcing independent reviewer contexts.
- Handling the absence of nested subagent support.

### 4. Preserve the original EGM methodology

No redesign of brainstorming, drafting, reviewing, or drift detection.

Only adapt the harness-specific execution instructions.

## How we know the integration works

I would require these tests before considering the integration complete:

| Test                      | Expected result                     |
| ------------------------- | ----------------------------------- |
| Invoke `@egm-brainstorm`  | Original EGM skill loads            |
| Read a persona            | Reads global `egm/agents/`          |
| Read a document standard  | Reads global `egm/resources/`       |
| Generate a concept brief  | Writes into the active project      |
| Launch Goldfish reviewers | Uses OpenCode subagents             |
| Run multiple reviewers    | Each receives an independent prompt |
| Review a design           | Preserves EGM review stages         |
| Switch projects           | Same global EGM installation works  |

I would also include an explicit isolation test using a unique marker visible only in the Elephant's conversation. Goldfish must not know that marker from inherited conversation history.

My conclusion: The right solution is a global EGM installation plus three OpenCode-specific integration files:

1. `opencode.json`
2. `agents/egm-goldfish.md`
3. `egm/docs/opencode.md`

This is a much more precise implementation plan than my earlier answer. It also avoids maintaining a fork of EGM's original skills and personas.