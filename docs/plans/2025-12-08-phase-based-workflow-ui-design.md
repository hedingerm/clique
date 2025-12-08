# Phase-Based Workflow UI Design

## Overview

Extend Clique to support the full BMAD methodology by adding a phase-based UI that reads `bmm-workflow-status.yaml` and presents workflows across all four BMAD phases.

## Problem

Currently Clique only supports Phase 3 (Implementation) by reading `sprint-status.yaml`. Users must manually run BMAD workflows for phases 0-2 without UI guidance.

## Solution

Add four phase-based tabs to the sidebar, each showing relevant workflow items with rich context and easy execution.

## UI Design

### Tab Structure

```
┌──────────────────────────────────────┐
│  Clique                          📁 🔄│
├──────────────────────────────────────┤
│ [Discovery] [Planning] [Solution] [Impl]│
├──────────────────────────────────────┤
│  (phase content here)                │
└──────────────────────────────────────┘
```

| Tab | Phase | Source | Workflows |
|-----|-------|--------|-----------|
| Discovery | 0 | `bmm-workflow-status.yaml` | brainstorm, research, product-brief |
| Planning | 1 | `bmm-workflow-status.yaml` | prd, validate-prd, ux-design |
| Solutioning | 2 | `bmm-workflow-status.yaml` | architecture, epics-stories, test-design, implementation-readiness |
| Implementation | 3 | `bmm-workflow-status.yaml` + `sprint-status.yaml` | sprint-planning + stories |

### Workflow Item Cards

Each workflow item displays as a rich card:

```
┌─────────────────────────────────────────┐
│ ▶ Product Brief                     [PM]│
│   Create comprehensive product brief    │
│   through collaborative discovery       │
└─────────────────────────────────────────┘
```

**Status icons:**
- `▶` (blue) - Next actionable item (play button visible)
- `✓` (green) - Completed (shows file path)
- `○` (gray) - Pending / required
- `⊘` (dim) - Skipped
- `◐` (yellow) - Conditional (waiting on prerequisites)

**Card content:**
- Line 1: Icon + Title + Agent badge (right-aligned)
- Line 2: Status value (file path, "skipped", "conditional", etc.)
- Line 3: Note from YAML (truncated if long)

### Detail Panel

Clicking a workflow item opens a side panel:

```
┌─────────────────────────────────────────────┐
│ Product Brief                           ✕   │
├─────────────────────────────────────────────┤
│ Phase:    Discovery (Phase 0)               │
│ Agent:    Analyst                           │
│ Command:  /bmad:bmm:workflows:product-brief │
│ Status:   Required                          │
├─────────────────────────────────────────────┤
│ Description                                 │
│ ─────────────────────────────────────────── │
│ Create comprehensive product briefs through │
│ collaborative step-by-step discovery.       │
├─────────────────────────────────────────────┤
│ Note                                        │
│ ─────────────────────────────────────────── │
│ (note from bmm-workflow-status.yaml)        │
├─────────────────────────────────────────────┤
│  [ Run Workflow ]    [ Mark Skipped ]       │
└─────────────────────────────────────────────┘
```

**Behavior:**
- "Run Workflow" → Opens terminal, runs command, shows toast with terminal link
- "Mark Skipped" → Updates `bmm-workflow-status.yaml` with `status: skipped`
- Description pulled from command file's frontmatter

### Welcome View (Empty State)

When no `bmm-workflow-status.yaml` exists:

```
┌─────────────────────────────────────────────┐
│            🚀 Welcome to Clique             │
│                                             │
│    Get started with the BMAD Method by      │
│    initializing your project workflow.      │
│                                             │
│         [ Initialize Workflow ]             │
│                                             │
│    This will run workflow-init to set up    │
│    your project's workflow status file.     │
└─────────────────────────────────────────────┘
```

Button runs `claude "/bmad:bmm:workflows:workflow-init"`.

### Implementation Tab

Combines workflow items and sprint stories:

```
┌─────────────────────────────────────────────┐
│ ◐ Sprint Planning                      [SM] │
│   conditional                               │
│   After implementation-readiness            │
└─────────────────────────────────────────────┘

── Sprint Stories ─────────────────────────────

▼ Epic 1: User Authentication
  ├─ ✓ 1-1-login-page
  ├─ ▶ 1-2-signup-flow          [ready-for-dev]
  └─ ○ 1-3-password-reset            [backlog]
```

If no `sprint-status.yaml`, shows: "Run Sprint Planning to create your first sprint"

## Code Architecture

```
src/
├── extension.ts              # Activation, registers all phases
├── core/
│   ├── workflowParser.ts     # Parse bmm-workflow-status.yaml
│   ├── sprintParser.ts       # Existing sprint-status.yaml parser
│   ├── types.ts              # Shared types (WorkflowItem, Phase, etc.)
│   └── fileWatcher.ts        # Unified file watching
├── phases/
│   ├── discovery/
│   │   ├── treeProvider.ts   # Phase 0 tree items
│   │   └── commands.ts       # Phase 0 specific commands
│   ├── planning/
│   │   ├── treeProvider.ts   # Phase 1 tree items
│   │   └── commands.ts
│   ├── solutioning/
│   │   ├── treeProvider.ts   # Phase 2 tree items
│   │   └── commands.ts
│   └── implementation/
│       ├── treeProvider.ts   # Phase 3 (stories from sprint-status)
│       └── commands.ts
├── ui/
│   ├── detailPanel.ts        # Webview for workflow details
│   └── welcomeView.ts        # Empty state / init view
└── workflowRunner.ts         # Existing terminal spawning logic
```

## Data Sources

### bmm-workflow-status.yaml

```yaml
workflow_status:
  - id: "product-brief"
    phase: 0
    status: "required"           # or file path, "skipped", "conditional"
    agent: "analyst"
    command: "product-brief"
    note: "Optional context"
```

### sprint-status.yaml

Existing format with epics and stories (unchanged).

## Key Behaviors

1. **File watching** - Extend dual-watcher pattern to also watch `bmm-workflow-status.yaml`
2. **Phase filtering** - `workflowParser.ts` filters items by phase number for each tab
3. **Backward compatible** - Existing sprint-status functionality preserved
4. **Status updates** - "Mark Skipped" writes back to `bmm-workflow-status.yaml`

## Out of Scope

- Phases beyond the four defined (prerequisite items shown in Discovery)
- Automatic phase progression/gating
- Multi-project support
