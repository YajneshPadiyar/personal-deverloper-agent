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


## Manual Copilot smoke-test checklist

A real Copilot subscription and an interactive VS Code are required for these
steps; they cannot be automated here.

- [ ] Install the `.vsix` via **Extensions: Install from VSIX...** and reload.
- [ ] Open Copilot Chat; type `@siebel` and confirm the participant and its six
      slash-commands appear with descriptions.
- [ ] With Copilot signed out, run any command and confirm the actionable
      "GitHub Copilot is required" message appears (no crash).
- [ ] Sign in to Copilot. Run `@siebel /generate-pm Generate a PM for the
      Account List Applet` and confirm:
  - [ ] Code streams into the chat.
  - [ ] The generated code is ES5 / AMD (`define()`, `typeof` guard,
        `SiebelJS.Namespace`, `SiebelJS.Extend`, no `let`/`const`/arrow
        functions).
  - [ ] A **Write customaccountlistpm.js to siebel/custom/** button appears.
  - [ ] Clicking it writes the file to `<customBasePath>/siebel/custom/` and the
        full path is reported.
- [ ] Re-run the same generation and confirm the modal overwrite prompt appears
      and that declining leaves the file unchanged.
- [ ] Open a Siebel JS file in the editor and run `@siebel /reviewer file`;
      confirm a severity summary table and a PASS / PASS WITH CONDITIONS / FAIL
      verdict are produced.
- [ ] In a git repo with pending changes run `@siebel /reviewer diff` and confirm
      the review uses the working-tree diff. In a folder with no repo, confirm it
      offers the active-file source instead.

## How it works

- **Participant:** `vscode.chat.createChatParticipant("siebel.openui", handler)`
  plus `contributes.chatParticipants` in `package.json`.
- **Model access:** `vscode.lm.selectChatModels({ vendor: "copilot" })` and
  `model.sendRequest` with `LanguageModelChatMessage`. The LM API has no system
  role, so the persona/system prompt is sent as a `User` message.
- **Grounding:** a static persona→docs baseline map plus keyword scoring, trimmed
  to `knowledgeTokenBudget` via `model.countTokens`, from the bundled
  `media/knowledge/` docs.
- **Standards:** a shared standards fragment is appended to every persona prompt.
