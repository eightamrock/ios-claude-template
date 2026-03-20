# Apple Platform Project Template for Claude Code

A self-contained project template that gives Claude Code full Apple platform SwiftUI development expertise out of the box. Copy this directory into any new project to get MVVM architecture guidance, HIG compliance, iOS 26 liquid glass patterns, multi-platform support, and pre-approved build permissions -- no configuration needed.

## What's Included

```
ios-claude-template/
  CLAUDE.md                                  # Project guidance with placeholders
  README.md                                  # This file
  setup.sh                                   # Interactive placeholder replacement script
  .gitignore                                 # Standard Apple development gitignore
  .mcp.json                                  # Xcode MCP server configuration
  .claude/
    settings.json                            # Permissions + MCP auto-approval
    skills/
      ios-swiftui-development/
        SKILL.md                             # Main skill (architecture, data flow, HIG, testing)
        references/
          swift-code-quality.md              # Concurrency, protocols, error handling, actors
          mvvm-observable-patterns.md        # @Observable MVVM, DI, composition, navigation
          swiftui-component-patterns.md      # Lists, forms, animation, search, async loading
          hig-compliance-checklist.md        # Full HIG compliance checklist (9 categories)
          liquid-glass-native-components.md  # Glass APIs, MeshGradient, native components
          swiftdata-persistence.md           # @Model, @Query, #Predicate, migrations
          app-intents-widgets.md             # AppIntent, WidgetKit, Live Activities, Control Center
          networking-api-patterns.md         # URLSession, API client, auth, pagination
          multiplatform-patterns.md          # macOS, watchOS, visionOS patterns, shared code
          notifications-background-tasks.md  # Push notifications, BGTaskScheduler, background downloads
```

### CLAUDE.md

Project-level guidance that Claude Code reads automatically. Covers build commands, MVVM architecture, SwiftUI best practices, iOS 26 liquid glass, Xcode MCP tools, code style, Human Interface Guidelines, LSP usage, Swift Testing, and SPM dependency management. Contains `{{PLACEHOLDER}}` markers for project-specific values.

### setup.sh

Interactive setup script that prompts for each placeholder value and replaces them in CLAUDE.md. Run once when setting up a new project.

### .mcp.json

Configures the **Xcode MCP server** (`mcpbridge`), giving Claude Code direct access to Xcode's build system and code intelligence. See [Xcode MCP Setup](#xcode-mcp-setup) below for prerequisites.

### .claude/settings.json

Pre-approves common development commands and Xcode MCP tools so Claude can build and test without prompting each time:
- `xcodebuild` (build, test)
- `swift build`, `swift test`, `swift package`, `swift format`
- `open *.xcodeproj` / `open *.xcworkspace`
- `xcrun`, `simctl`
- All Xcode MCP tools (`mcp__xcode__*`)

### .claude/skills/ios-swiftui-development/

A custom skill with 10 supporting reference files. Claude Code discovers this automatically when working in the project. It provides deep knowledge of:
- MVVM with `@Observable` (patterns, composition, dependency injection)
- SwiftUI components (lists, grids, forms, navigation, animation, search)
- Swift code quality (naming, concurrency, actors, Sendable, error handling)
- Apple Human Interface Guidelines (44pt targets, Dynamic Type, Dark Mode, accessibility)
- iOS 26 liquid glass (automatic vs manual glass, MeshGradient, native components)
- SwiftData persistence (@Model, @Query, #Predicate, migrations, testing)
- App Intents & Widgets (AppIntent, WidgetKit, Live Activities, Control Center widgets)
- Networking (URLSession async/await, API client pattern, auth, pagination)
- Multi-platform (macOS sidebar, watchOS, visionOS, shared code strategies)
- Push notifications & background tasks (UNUserNotificationCenter, BGTaskScheduler)

## Setup

### 1. Copy the template

```bash
cp -r ios-claude-template/ /path/to/your/new-project/
```

Or copy just the Claude Code files into an existing project:

```bash
cp ios-claude-template/CLAUDE.md /path/to/existing-project/
cp ios-claude-template/.gitignore /path/to/existing-project/
cp -r ios-claude-template/.claude /path/to/existing-project/
```

### 2. Run setup script (recommended)

```bash
cd /path/to/your/project
./setup.sh
```

The script will prompt for your project name, bundle ID, platforms, and other values, then replace all `{{PLACEHOLDER}}` markers in CLAUDE.md.

### 3. Or replace placeholders manually

Open `CLAUDE.md` and replace all `{{PLACEHOLDER}}` markers:

| Placeholder | Replace With | Example |
|-------------|-------------|---------|
| `{{PROJECT_NAME}}` | Your Xcode project/scheme name | `MyApp` |
| `{{BUNDLE_ID}}` | Your app's bundle identifier | `com.example.myapp` |
| `{{PROJECT_DESCRIPTION}}` | One-line description of your app | `A recipe organizer with meal planning` |
| `{{APP_TYPE}}` | The type of project | `SwiftUI app`, `widget`, `SPM package` |
| `{{PLATFORMS}}` | Target platforms | `iOS 26, macOS 26, watchOS 12` |
| `{{KEY_FILES_DESCRIPTION}}` | Brief note about important files | `Core data models and API layer` |

### 4. Start Claude Code

```bash
cd /path/to/your/project
claude
```

Claude Code automatically picks up:
- `CLAUDE.md` for project guidance
- `.claude/settings.json` for permissions
- `.claude/skills/ios-swiftui-development/` for platform expertise

No manual configuration or skill installation required.

## Xcode MCP Setup

The template includes an `.mcp.json` that configures the [Xcode MCP server](https://developer.apple.com/documentation/Xcode/giving-agentic-coding-tools-access-to-xcode), giving Claude Code direct access to Xcode's build system and code intelligence tools via the Model Context Protocol.

### Prerequisites

1. **Xcode 26+** must be installed
2. **Enable MCP connections in Xcode:** Open Xcode > Settings > General > check **"Allow MCP Connections"**
3. **Xcode must be running** when you start Claude Code (the MCP bridge connects to a running Xcode instance)

### How It Works

The `.mcp.json` at the project root tells Claude Code to launch `xcrun mcpbridge`, which acts as a STDIO bridge between Claude Code and Xcode's MCP tool service. It reads JSON-RPC 2.0 messages from stdin and forwards responses to stdout.

```json
{
  "mcpServers": {
    "xcode": {
      "command": "xcrun",
      "args": ["mcpbridge"]
    }
  }
}
```

### Available Xcode MCP Tools

Once connected, Claude Code gains access to Xcode tools prefixed with `mcp__xcode__`, including:

| Tool | Description |
|------|-------------|
| `BuildProject` | Build the project/scheme in Xcode |
| `GetBuildLog` | Retrieve build logs from the last build |
| `XcodeListNavigatorIssues` | Get build errors, warnings, and analyzer issues |
| `GetTestList` | List available tests in the project |
| `RunAllTests` | Run all tests in the project |
| `RunSomeTests` | Run specific test classes or methods |
| `RenderPreview` | Render a SwiftUI preview for visual validation |
| `DocumentationSearch` | Search Apple developer documentation |
| `ExecuteSnippet` | Run a Swift code snippet |
| `XcodeRefreshCodeIssuesInFile` | Get diagnostics for a specific source file |
| `XcodeRead` | Read file contents via Xcode |
| `XcodeWrite` | Write a new file via Xcode |
| `XcodeUpdate` | Update an existing file via Xcode |
| `XcodeGrep` | Search file contents in the project |
| `XcodeGlob` | Find files by name pattern |
| `XcodeLS` | List directory contents |
| `XcodeMV` | Move/rename files |
| `XcodeRM` | Remove files |
| `XcodeMakeDir` | Create directories |
| `XcodeListWindows` | List open Xcode windows |

These tools are pre-approved in `.claude/settings.json` via `"mcp__xcode__*"`.

### Multiple Xcode Instances

If you have multiple Xcode instances running, `mcpbridge` uses this fallback logic:
1. If exactly one Xcode process is running, it connects to that one
2. If multiple are running, it uses `xcode-select` to determine which to connect to
3. If none are running, it exits with an error

To target a specific instance, set the `MCP_XCODE_PID` environment variable:

```json
{
  "mcpServers": {
    "xcode": {
      "command": "xcrun",
      "args": ["mcpbridge"],
      "env": {
        "MCP_XCODE_PID": "12345"
      }
    }
  }
}
```

### Troubleshooting

- **"MCP server xcode failed to start"** -- Make sure Xcode is running and MCP connections are enabled in Xcode > Settings > General
- **Wrong Xcode instance** -- Set `MCP_XCODE_PID` or use `xcode-select -s` to point to the correct Xcode
- **Tools not appearing** -- Restart Claude Code after enabling MCP in Xcode; the MCP server list refreshes on session start

## Requirements

- **Xcode 26+** with Swift 6
- **Swift Package Manager** for all dependencies (no CocoaPods/Carthage)
- **swift-lsp** Claude Code plugin (`claude plugins install swift-lsp`)

## Standards Enforced

| Area | Standard |
|------|----------|
| Architecture | MVVM with `@Observable`, thin views, protocol-defined services |
| Data flow | `@State`, `@Binding`, `@Bindable`, `@Environment` decision guide |
| UI framework | SwiftUI, `NavigationStack`, `TabView` |
| Design | Apple HIG, SF Symbols, Dynamic Type, Dark Mode, 44pt touch targets |
| iOS 26 | Liquid glass on system chrome, `.glassEffect` for custom floats, `MeshGradient` |
| Persistence | SwiftData (`@Model`, `@Query`, `#Predicate`) |
| Networking | URLSession async/await, protocol-based API clients |
| Intents | App Intents, WidgetKit, Live Activities |
| Concurrency | `async/await`, `@MainActor`, `actor`, `Sendable` |
| Testing | Swift Testing framework (`@Test`, `#expect`), protocol mocks |
| Code style | Swift API Design Guidelines, no force-unwraps, `guard` for early exits |
| Dependencies | SPM only, prefer Apple frameworks |
| LSP | Diagnostics required after edits, all warnings resolved before commit |
| Multi-platform | `#if os()` conditionals, platform-adaptive views, shared SPM modules |

## Customization

### Adding project-specific permissions

Edit `.claude/settings.json` to allow additional commands:

```json
{
  "permissions": {
    "allow": [
      "Bash(xcodebuild *)",
      "Bash(swift build*)",
      "Bash(your-custom-command*)"
    ]
  }
}
```

### Modifying the skill

Edit files in `.claude/skills/ios-swiftui-development/` directly. Changes are picked up on the next Claude Code session. The skill is project-local, so edits only affect this project.

### Removing sections

If a section in `CLAUDE.md` doesn't apply (e.g., liquid glass for an iOS 17 target), delete it. Claude Code reads whatever is present.
