# Siebel Open UI Copilot

A VS Code **GitHub Copilot Chat** extension that reproduces the Siebel Open UI
code-generation and code-review capabilities of the Kiro workspace. It adds a
single chat participant, `@siebel`, with one slash-command per Siebel persona.
The Siebel knowledge base ships inside the extension, so Copilot always has the
correct domain context with no network calls.

This is an **internal team tool**. It uses your existing GitHub Copilot
subscription through the VS Code Language Model API — there is no separate API
key, no model hosting, and no backend.

> The code this extension *generates* follows the project's ES5 / AMD Siebel
> conventions. The extension's own source is modern TypeScript; the ES5 rule
> does not apply to it.

## Prerequisites

- **VS Code 1.90.0 or later** (required for the Chat and Language Model APIs this
  extension uses — see `engines.vscode` in `package.json`).
- **GitHub Copilot** and **GitHub Copilot Chat** extensions installed and signed
  in with an active Copilot subscription. The extension talks to Copilot through
  `vscode.lm`; if no Copilot model is available it shows an actionable message
  instead of failing silently.

## Installation (Install from VSIX)

1. Build the package (see *Building*) or obtain `siebel-openui-copilot-<version>.vsix`.
2. In VS Code open the Command Palette (`Ctrl+Shift+P`) and run
   **Extensions: Install from VSIX...**.
3. Select the `.vsix` file.
4. Reload VS Code when prompted.
5. Open the Copilot Chat panel and type `@siebel` to confirm the participant
   appears.

## Commands

Invoke the participant with `@siebel`, optionally followed by a slash-command.
With no slash-command, requests route to the **developer** persona.

| Command | Persona | What it does |
|---|---|---|
| `@siebel /developer` | Open UI Developer | Generates PM/PR/PW **and CR** code; general Open UI help. |
| `@siebel /reviewer` | Open UI Reviewer | Siebel-aware code review of the active file or the git diff. |
| `@siebel /generate-pm` | PM Generator | Generates a Presentation Model. |
| `@siebel /generate-pr` | PR Generator | Generates a Physical Renderer. |
| `@siebel /architect` | Open UI Architect | Solution design and impact analysis. |
| `@siebel /manifest` | Manifest Specialist | Manifest Administration configuration. |

Each persona keeps its own system prompt and output format (ported from the
`.kiro/agents/*.md` definitions); behaviors are not blended.

### Usage examples

Generate a Presentation Model:

```
@siebel /generate-pm Generate a PM for the Account List Applet (list applet, Account business component) that validates the Name field before WriteRecord.
```

Generate a Physical Renderer:

```
@siebel /generate-pr Render a status badge column in the Opportunity List Applet based on the Sales Stage field.
```

Full-stack development (may emit PM + PR + CR):

```
@siebel /developer Build a custom content renderer for the Contact Form Applet that lays out the record as a card.
```

Review the file you are editing:

```
@siebel /reviewer file Check this PM for lifecycle and upgrade-safety issues.
```

Review your pending git changes:

```
@siebel /reviewer diff
```

If you do not state `file` or `diff`, the extension uses the
`siebelOpenUI.defaultReviewSource` setting, prompting you to choose when that is
set to `ask`.

### Writing generated code to disk

When a generation command produces Siebel code, the code is always shown inline
in the chat. The response also offers a **Write &lt;filename&gt; to siebel/custom/**
button for each recognized artifact. Clicking it writes the file under
`<customBasePath>/siebel/custom/` using the project naming conventions
(`Custom<Name><Type>` class, lowercase `custom<name><type>.js` on disk). If the
target file already exists you are asked to confirm before it is overwritten;
files are never overwritten silently.

## Settings

| Setting | Default | Purpose |
|---|---|---|
| `siebelOpenUI.customBasePath` | `""` (workspace root) | Base folder under which `siebel/custom/` is resolved for generated artifacts. |
| `siebelOpenUI.defaultReviewSource` | `"ask"` | Default review input source: `ask`, `activeFile`, or `gitDiff`. |
| `siebelOpenUI.knowledgeTokenBudget` | `12000` | Maximum tokens of bundled knowledge grounding supplied per request. |
| `siebelOpenUI.model` | `"auto"` | Preferred Copilot model family; `auto` lets VS Code choose. |

## Project custom rules (`.github/siebel/`)

A project can extend the built-in coding standards, review rules, and security
guardrails with its own markdown **custom rule files**, checked in under
`.github/siebel/` at the workspace root (that folder or any nested subfolder is
scanned recursively). When present, these files are injected **in full** and
**take precedence** over the built-in knowledge on conflict; the built-ins stay
as a fallback for anything the custom files do not cover. When no custom files
exist, the extension behaves exactly as before.

Files are discovered only in the **primary** (first) workspace folder's
`.github/siebel/` tree. This location is used because the rules are specific to
the Copilot extension integration and version naturally alongside other
`.github` configuration.

### Filename scheme

Each file belongs to a **category** (`standards`, `review`, or `security`) and a
**sub-category** (`js` or `siebel`). The sub-category is encoded in the filename;
Siebel files additionally encode the target artifact type:

```
JS:     custom-<category>-js-<label>.md
Siebel: custom-<category>-siebel-<type>-<label>.md     <type> ∈ pm | pr | pw | cr | any
```

- `<label>` is free descriptive text (naming only).
- The prefix and the `<type>` segment are matched **case-insensitively**.
- A `siebel` file with an unrecognized `<type>` is treated as `any` and a warning
  is logged.
- Files that do not match the scheme, or that sit outside `.github/siebel/`, are
  ignored.

Examples:

```
.github/siebel/custom-standards-js-es5.md
.github/siebel/custom-standards-siebel-any-naming.md
.github/siebel/custom-review-siebel-pm-lifecycle.md
.github/siebel/custom-security-js-xss.md
```

### Applicability

- **JS rules always apply**, for every persona and artifact type.
- **Siebel rules apply by artifact type.** The type is detected from the
  generation command, then the Siebel base class the code extends
  (`PresentationModel` → PM, `PhysicalRenderer` → PR, the plug-in-wrapper base →
  PW, the content-renderer base → CR), then the filename (`...pm.js` → PM, etc.).
  A Siebel rule is included when its `<type>` is `any` or matches the detected
  type. When the type cannot be determined, a manual `/reviewer` asks you how to
  proceed (pick a type, apply all Siebel rules, or skip them), while an
  auto-review applies all rules.
- Generation and advisory flows use the **standards** and **security** categories
  and list the applied files in a short note. The **review** category is used
  only by `/reviewer`.

### Stepped review

`/reviewer` runs as an ordered sequence of **named steps** — category order
**standards → review → security**, and within each category **JS before Siebel**.
Each applicable custom rule file is its own step, labeled by the file's first
`# H1` heading (or its filename when there is no heading), and ordered
alphabetically. The built-in JS and Siebel review rules also run as their own
steps, so reviews work with zero custom files. After every step, a final
aggregating step produces the severity summary table and the overall
PASS / PASS WITH CONDITIONS / FAIL verdict.

Sample rule files you can copy are in `docs/examples/github-siebel/`.

## Contributing / development

Building from source, running the unit and integration suites, the manual
Copilot smoke-test checklist, and an architecture overview live in a separate
developer guide:
[`DEVELOPMENT.md`](https://github.com/YajneshPadiyar/siebel-openui/blob/main/siebel-openui-copilot/DEVELOPMENT.md).
That file is not shipped inside the `.vsix`; it lives in the repository.
