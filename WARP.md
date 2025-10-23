# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

Fabric is an open-source Go-based framework for augmenting humans using AI. It organizes prompts (called "Patterns") by real-world tasks and provides a CLI interface, REST API, and web interface for interacting with multiple AI providers.

## Common Development Commands

### Building and Running

```bash
# Build the main fabric binary
go build -o fabric ./cmd/fabric

# Run fabric directly
./fabric --setup
./fabric --help

# Build other binaries
go build -o to_pdf ./cmd/to_pdf
go build -o code_helper ./cmd/code_helper
go build -o generate_changelog ./cmd/generate_changelog
```

### Testing

```bash
# Run all tests
go test ./...

# Run tests for specific package
go test ./internal/cli
go test ./internal/core

# Run tests with coverage
go test -cover ./...
```

### Installation

```bash
# Install from source (installs to $GOPATH/bin)
go install github.com/danielmiessler/fabric/cmd/fabric@latest

# First-time setup (required after installation)
fabric --setup
```

### Pattern Development

```bash
# List all patterns
fabric --listpatterns

# Update patterns from repository
fabric --updatepatterns

# Test a pattern
echo "test input" | fabric --pattern extract_wisdom

# Test with streaming output
echo "test input" | fabric --stream --pattern summarize
```

### Web Interface

```bash
# Start the REST API server (required for web UI)
fabric --serve

# In a separate terminal, from the web/ directory:
cd web
npm run dev        # or: pnpm run dev

# Access web UI at http://localhost:5173
# API is available at http://localhost:8080
```

## Pull Request Requirements

After opening a PR, you MUST generate a changelog entry:

```bash
cd cmd/generate_changelog
go build -o generate_changelog .
./generate_changelog --incoming-pr YOUR_PR_NUMBER

# Optional flags:
# --ai-summarize    Generate AI-enhanced summaries
# --push           Automatically push the changelog commit
```

**Requirements:**
- PR must be open and mergeable (no conflicts)
- Working directory must be clean
- GitHub token available (GITHUB_TOKEN env var)

## Code Architecture

### High-Level Structure

```
fabric/
├── cmd/                    # Executable commands
│   ├── fabric/            # Main CLI application
│   ├── generate_changelog/# Automated changelog generator
│   ├── code_helper/       # Code analysis helper
│   └── to_pdf/            # LaTeX to PDF converter
├── internal/              # Private application packages
│   ├── cli/              # CLI interface and flag handling
│   ├── core/             # Core plugin registry and initialization
│   ├── chat/             # Chat session management
│   ├── plugins/          # AI vendor integrations (OpenAI, Anthropic, etc.)
│   ├── server/           # REST API server
│   ├── tools/            # Utilities (YouTube, converters, etc.)
│   ├── i18n/             # Internationalization support
│   └── log/              # Debug logging
├── data/
│   ├── patterns/         # AI prompt patterns (Markdown files)
│   └── strategies/       # Prompt strategies (JSON files)
├── web/                  # Svelte-based web interface
└── scripts/              # Installation and utility scripts
```

### Key Architectural Patterns

**Plugin Registry Pattern**: The `internal/core` package manages a central registry (`PluginRegistry`) that initializes and provides access to:
- AI vendor plugins (OpenAI, Anthropic, Ollama, Gemini, etc.)
- Database connections
- Configuration management
- Tool integrations (YouTube, web scraping)

**CLI Flow**: `cmd/fabric/main.go` → `internal/cli/cli.go` which follows this sequence:
1. Initialize flags and internationalization
2. Initialize plugin registry and database
3. Handle setup/server commands
4. Handle configuration commands
5. Handle listing commands
6. Process input (stdin, files, YouTube, etc.)
7. Execute chat/pattern processing

**Pattern System**: Patterns are Markdown files stored in `data/patterns/[pattern-name]/system.md` with this structure:
- `# IDENTITY and PURPOSE` - Define the AI's role
- `# STEPS` - Execution steps
- `# OUTPUT` - Expected output format
- `# EXAMPLE` - Sample output

**Vendor Abstraction**: All AI providers implement a common interface in `internal/plugins/ai/`, allowing seamless switching between OpenAI, Anthropic, Ollama, Gemini, etc.

### Configuration

User configuration is stored in `~/.config/fabric/`:
- `.env` - API keys and credentials
- `db/` - SQLite database for contexts/sessions
- `patterns/` - Built-in patterns (updated via git)
- Custom patterns can be stored in a user-defined directory

## Development Guidelines

### Code Style

- Follow standard Go conventions (`gofmt`, `golint`)
- Use meaningful variable and function names
- Keep functions focused and small
- Write tests for new functionality

### Commit Messages

Use conventional commit format:
```
feat: add new pattern for code analysis
fix: resolve OAuth token refresh issue
docs: update installation instructions
```

### Project-Specific Patterns

**Flag Handling**: Use `go-flags` library. All CLI flags are defined in `internal/cli/flags.go`

**Internationalization**: Use the `i18n` package. All user-facing strings should use `i18n.T("key")` for translation support.

**Error Handling**: Return errors up the stack; log at appropriate levels using `debuglog.Log()`.

**AI Provider Integration**: When adding a new AI provider:
1. Implement the vendor interface in `internal/plugins/ai/[vendor]/`
2. Register it in the plugin registry initialization
3. Add vendor-specific configuration to the setup flow

### Testing AI Patterns

To test changes with actual AI providers:
```bash
# Test with specific model
fabric --model gpt-4 --pattern summarize < input.txt

# Test with different vendors
fabric --vendor "OpenAI" --model gpt-4 --pattern extract_wisdom < article.txt

# Test streaming output
fabric --stream --pattern analyze_claims < claims.txt

# Test YouTube integration
fabric -y "https://youtube.com/watch?v=..." --pattern extract_wisdom
```

## Web Development

The web interface (`web/`) is a separate Svelte application:

```bash
# Install dependencies
cd web
./scripts/pnpm-install.sh  # or npm-install.sh

# Development server
pnpm run dev  # or npm run dev

# Build for production
pnpm run build
```

**Important**: The web UI requires the Fabric REST API server to be running (`fabric --serve`) on port 8080.

## Helper Tools

### `generate_changelog`
Automated changelog generation for PRs and releases. Required for all PRs.

### `code_helper`
Generates JSON representation of code directories for AI analysis. Used with the `create_coding_feature` pattern.

### `to_pdf`
Converts LaTeX to PDF. Works with the `write_latex` pattern.

## Environment Variables

Set in `~/.bashrc`, `~/.zshrc`, or system environment:

```bash
# Required for Go development
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$GOPATH/bin:$GOROOT/bin:$HOME/.local/bin:$PATH

# Optional: Fabric configuration
export FABRIC_MODEL_PATTERN_NAME=vendor|model  # Per-pattern model mapping
export FABRIC_ALIAS_PREFIX=fab-  # Prefix for pattern aliases
```

## Common Issues

**"fabric: command not found"**: Add `$GOPATH/bin` to your PATH.

**Pattern not found**: Run `fabric --updatepatterns` to sync latest patterns from repository.

**AI vendor errors**: Check API keys in `~/.config/fabric/.env` and ensure the vendor is properly configured via `fabric --setup`.

**Web UI can't connect**: Ensure `fabric --serve` is running on port 8080 before starting the web development server.
