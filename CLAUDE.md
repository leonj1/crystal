# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Crystal - Multi-Session Claude Code Manager

Crystal is an Electron desktop application for managing multiple Claude Code instances against a single directory using git worktrees. It enables running parallel Claude Code sessions with different approaches to the same problem.

## Build and Development Commands

```bash
# One-time setup (installs dependencies, builds main process, rebuilds native modules)
pnpm run setup

# Development (runs Electron app with hot reload)
pnpm run dev    # or pnpm electron-dev

# Build commands
pnpm build      # Build all components
pnpm build:mac  # Build macOS app (only works on macOS)
pnpm build:linux # Build Linux app

# Testing
pnpm test       # Run Playwright E2E tests
pnpm test:ui    # Run tests with UI
pnpm typecheck  # TypeScript type checking
pnpm lint       # ESLint linting

# Run a specific test
pnpm test tests/permissions.spec.ts
```

## High-Level Architecture

Crystal uses a modular monorepo structure with pnpm workspaces:

### Frontend (React + TypeScript)
- **State Management**: Zustand stores (`sessionStore`, `errorStore`) with targeted updates
- **Terminal**: XTerm.js with 50,000 line scrollback buffer
- **UI Components**: Radix UI primitives with Tailwind CSS
- **Key Hook**: `useSessionView.ts` (941 lines) - Extracted session view logic

### Main Process (Electron + Node.js)
Recently refactored from monolithic structure into modular components:

- **Core Modules**:
  - `index.ts` (414 lines): Electron setup and initialization
  - `ipc/git.ts` (843 lines): Git operation handlers
  - `ipc/session.ts` (428 lines): Session management handlers
  - `events.ts` (359 lines): Event coordination

- **Service Layer**:
  - `SessionManager`: Manages Claude Code lifecycle
  - `ClaudeCodeManager`: Spawns processes via node-pty
  - `WorktreeManager`: Git worktree operations
  - `DatabaseService`: SQLite with Better-SQLite3
  - `TaskQueue`: Bull queue for async operations

### Database Schema
SQLite database with migration system:
- `projects`: Project configurations
- `sessions`: Session metadata and status
- `session_outputs`: Terminal output (raw JSON + formatted)
- `conversation_messages`: Claude conversation history
- `execution_diffs`: Git diffs per execution
- `prompt_markers`: Navigation with timestamps

### Communication Architecture
- **IPC**: Secure contextBridge for renderer-main communication
- **Events**: Real-time updates for session status changes
- **WebSockets**: Development mode only for backwards compatibility

## Critical Implementation Details

### ⚠️ Session Output Handling (FRAGILE - DO NOT MODIFY)
The session output system is complex and modifications frequently cause issues. Key points:
- Database stores raw JSON messages as-is
- Frontend transforms JSON to formatted output on-the-fly
- Real-time streaming supplements but doesn't replace database data
- Uses `requestAnimationFrame` and mutex locks for timing
- Never clear `formattedOutput` state on session switch

### Timestamp Handling
All timestamps require special handling for timezone consistency:
- Database stores UTC using SQLite's `datetime()`
- Use provided utilities: `parseTimestamp()`, `formatDuration()`, `getTimeDifference()`
- Never parse SQLite timestamps directly with `new Date()`
- Always append 'Z' when selecting timestamps from database

### State Management Patterns
Crystal uses targeted updates to minimize re-renders:
- Never reload entire lists when one item changes
- Use state setter callbacks for latest state
- Handle IPC events for real-time synchronization
- Database is the single source of truth

### Git Worktree Integration
Each session runs in an isolated git worktree:
- Automatic branch creation and management
- Support for rebase, squash, and merge operations
- Handles empty repos with initial commit
- Cleanup on session deletion

## Development Patterns

### Adding New Features
1. Check existing patterns in similar components
2. Use existing libraries (check package.json first)
3. Follow established IPC communication patterns
4. Add database migrations for schema changes
5. Use targeted state updates, not global refreshes

### Common Tasks
- **Add IPC Handler**: Create in appropriate `ipc/*.ts` file, register in `index.ts`
- **Add Database Table**: Create migration in `migrations/`, update types
- **Add UI Component**: Use Radix UI primitives, style with Tailwind
- **Handle Session Updates**: Use IPC events, update state targetedly

### Testing Considerations
- Playwright tests run against dev server (port 4521)
- Test permissions and MCP integration carefully
- Screenshots saved on test failure
- Use `--headed` flag for debugging tests

## Special Considerations

1. **Native Modules**: `better-sqlite3` and `node-pty` require rebuilding for Electron
2. **Process Management**: Claude Code instances managed via node-pty PTY processes
3. **Conversation Continuation**: Full history loaded from database
4. **Diff Viewer CSS**: Keep styling simple, avoid complex overrides
5. **Performance**: Lazy load outputs, debounce updates, use caching

## Debugging Tips
- Enable verbose logging in Settings
- Check DevTools console for frontend errors
- Main process logs to terminal
- Use Messages view to inspect raw Claude output
- Database located at `~/.crystal/crystal.db`