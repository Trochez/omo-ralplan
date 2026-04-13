# Configuration Guide

This guide covers detailed configuration for OMO-Ralplan.

## Oh-My-OpenCode Configuration

### Required Agents

```json
{
  "agents": {
    "oracle": {
      "mode": "subagent",
      "category": "consultant",
      "tools": {
        "read": true,
        "grep": true,
        "glob": true
      }
    },
    "explore": {
      "mode": "subagent",
      "category": "search",
      "tools": {
        "read": true,
        "grep": true,
        "glob": true
      }
    },
    "librarian": {
      "mode": "subagent",
      "category": "research",
      "tools": {
        "webfetch": true,
        "websearch": true,
        "read": true
      }
    },
    "metis": {
      "mode": "subagent",
      "category": "analyst",
      "tools": {
        "read": true,
        "grep": true,
        "glob": true
      }
    },
    "momus": {
      "mode": "subagent",
      "category": "reviewer",
      "tools": {
        "read": true,
        "grep": true,
        "glob": true
      }
    }
  }
}
```

### Required Categories

```json
{
  "categories": {
    "visual-engineering": {
      "description": "Frontend, UI/UX, design, styling, animation"
    },
    "ultrabrain": {
      "description": "Hard logic, architecture, algorithms"
    },
    "deep": {
      "description": "Autonomous research and implementation"
    },
    "quick": {
      "description": "Trivial tasks, single file changes"
    },
    "artistry": {
      "description": "Creative, unconventional solutions"
    },
    "writing": {
      "description": "Documentation, prose, technical writing"
    }
  }
}
```

## Environment Variables

### Timeout Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `OMX_CONSENSUS_AGENT_TIMEOUT_MS` | 120000 | Per-agent call timeout (2 minutes) |
| `OMX_CONSENSUS_TOTAL_TIMEOUT_MS` | 600000 | Total workflow timeout (10 minutes) |
| `OMX_ASK_USER_TIMEOUT_MS` | 300000 | User response timeout (5 minutes) |
| `OMX_CONSENSUS_MAX_REVIEW_ITERATIONS` | 5 | Max re-review loops |
| `OMX_CONSENSUS_CIRCUIT_BREAKER_THRESHOLD` | 3 | Same error recurrence limit |

### Setting Environment Variables

```bash
# In ~/.bashrc or ~/.zshrc
export OMX_CONSENSUS_AGENT_TIMEOUT_MS=300000
export OMX_CONSENSUS_TOTAL_TIMEOUT_MS=900000
export OMX_CONSENSUS_MAX_REVIEW_ITERATIONS=3
```

### Timeout Recommendations

| Scenario | Agent Timeout | Total Timeout | Max Iterations |
|----------|---------------|---------------|----------------|
| Quick tasks | 60000 | 300000 | 3 |
| Standard tasks | 120000 | 600000 | 5 |
| Complex tasks | 300000 | 900000 | 5 |
| Critical tasks | 600000 | 1800000 | 7 |

## Model Configuration

OMO-Ralplan is model-agnostic. Configure models in `oh-my-opencode.json`:

```json
{
  "agents": {
    "oracle": {
      "model": "your-preferred-model",
      "reasoningEffort": "high",
      "mode": "subagent",
      "category": "consultant"
    }
  }
}
```

### Reasoning Effort Recommendations

| Task | Recommended Effort |
|------|-------------------|
| Architecture review | high/xhigh |
| Quick search | minimal/low |
| Complex logic | high |
| UI/UX work | medium/high |

## Skill Installation

### Method 1: Direct Copy

```bash
mkdir -p ~/.agents/skills/omo-ralplan
cp SKILL.md ~/.agents/skills/omo-ralplan/
cp README.md ~/.agents/skills/omo-ralplan/
```

### Method 2: Git Clone

```bash
git clone https://github.com/your-username/omo-ralplan.git
cd omo-ralplan
mkdir -p ~/.agents/skills/omo-ralplan
cp SKILL.md README.md ~/.agents/skills/omo-ralplan/
```

### Method 3: Symlink

```bash
git clone https://github.com/your-username/omo-ralplan.git
ln -s $(pwd)/omo-ralplan/SKILL.md ~/.agents/skills/omo-ralplan/SKILL.md
```

## Verification

### Check Installation

```bash
ls -la ~/.agents/skills/omo-ralplan/
# Should show:
# SKILL.md
# README.md
```

### Test Skill

```
$omo-ralplan "Test planning workflow"
```

### Verify Configuration

```bash
# Check oh-my-opencode.json
cat ~/.config/oh-my-opencode/oh-my-opencode.json | grep -A5 "oracle"

# Check environment variables
env | grep OMX_CONSENSUS
```

## Troubleshooting Configuration

### Issue: Agent Not Found

**Error:** `oracle subagent unavailable`

**Solution:**
```json
{
  "agents": {
    "oracle": {
      "mode": "subagent",
      "category": "consultant"
    }
  }
}
```

### Issue: Timeout Too Short

**Error:** Workflow hangs at Architect step

**Solution:**
```bash
export OMX_CONSENSUS_AGENT_TIMEOUT_MS=300000
```

### Issue: Category Not Found

**Error:** `Category 'visual-engineering' not found`

**Solution:**
```json
{
  "categories": {
    "visual-engineering": {}
  }
}
```

### Issue: Skill Not Recognized

**Error:** `$omo-ralplan: command not found`

**Solution:**
```bash
# Verify skill location
ls ~/.agents/skills/omo-ralplan/SKILL.md

# Check frontmatter
head -5 ~/.agents/skills/omo-ralplan/SKILL.md
```
