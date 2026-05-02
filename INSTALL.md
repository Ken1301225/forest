# Forest Installation

Copy the skill and agent files to the appropriate OpenCode locations.

## Step 1: Install the Skill

```bash
mkdir -p ~/.agents/skills
cp -r skills/forest-orchestration ~/.agents/skills/
```

## Step 2: Install the Agents

```bash
mkdir -p ~/.config/opencode/agents
cp agents/forest*.md ~/.config/opencode/agents/
```

## Step 3: (Optional) Configure Model Options

Forest's subagents benefit from model-specific configuration. Add to `~/.config/opencode/opencode.json`:

```json
"provider": {
  "opencode-go": {
    "models": {
      "deepseek-v4-pro": {
        "options": {
          "thinking": { "type": "enabled" },
          "reasoningEffort": "max"
        }
      }
    }
  }
}
```

Other models (kimi, qwen, glm, etc.) may also support `thinking`/`reasoningEffort` — verify each model's capabilities before adding options.

## Step 4: Verify

1. Start a new OpenCode session
2. Press **Tab** to switch to the **Forest** agent
3. Ask a question like: "Analyze this project for potential bugs"
4. Forest should load the orchestration skill, ask you to select subagents and rounds, then begin deliberation

## One-Liner Install

```bash
git clone https://github.com/Ken1301225/forest /tmp/forest && \
mkdir -p ~/.agents/skills ~/.config/opencode/agents && \
cp -r /tmp/forest/skills/forest-orchestration ~/.agents/skills/ && \
cp /tmp/forest/agents/forest*.md ~/.config/opencode/agents/ && \
rm -rf /tmp/forest && \
echo "Forest installed. Start OpenCode and switch to the Forest agent (Tab)."
```

## Uninstall

```bash
rm -rf ~/.agents/skills/forest-orchestration
rm ~/.config/opencode/agents/forest*.md
```
