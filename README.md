# Open Cowork

> **Forked from [Kimi CLI](https://github.com/MoonshotAI/kimi-cli)** - A Windows desktop application similar to Claude Cowork

**Open Cowork** is an AI-powered desktop application for Windows that brings Claude Cowork-like functionality to your local machine. Built on the foundation of Kimi CLI, it provides a graphical interface for AI-assisted coding and file management without requiring terminal knowledge.

## About This Project

This project is forked from the excellent [Kimi CLI](https://github.com/MoonshotAI/kimi-cli) by Moonshot AI. While Kimi CLI focuses on terminal-based workflows, Open Cowork transforms it into a desktop application with:
- **Windows-native desktop UI** - No terminal required
- **Folder-based workspace management** - Point and click to authorize folders
- **Visual task orchestration** - See the AI's work progress in real-time
- **Similar to Claude Cowork** - Familiar workflow for desktop users

> [!IMPORTANT]
> Open Cowork is under active development. This is an independent fork and is not officially affiliated with Moonshot AI or the original Kimi CLI project.

## Getting Started

> [!NOTE]
> Open Cowork for Windows is currently under development. Check back soon for installation instructions.

**Planned Features:**
- **Folder-based workflows**: Point the app at specific directories for AI-assisted work
- **Graphical interface**: No terminal required - interact through a familiar desktop UI
- **Project management**: Manage multiple workspace folders and sessions
- **Background execution**: Delegate tasks and continue working while the AI processes
- **Built on proven technology**: Leverages Kimi CLI's powerful agent architecture

## Key Features (Planned)

### Desktop Application

**Folder access with visual interface**: Work with specific folders on your computer through a graphical interface. The AI agent can:
- Read and organize files within authorized folders
- Create and modify documents, spreadsheets, and code files
- Execute multi-step workflows across your file system
- Manage projects with visual task tracking

**Orchestration UI**: See the AI's work progress in real-time:
- View active steps and tool usage
- Track sub-agent spawning and parallel task execution
- Monitor file changes and command outputs
- Approve or reject actions through dialog prompts

**Session management**: 
- Create and switch between multiple workspace sessions
- Resume previous work automatically
- Keep work isolated per project folder

## Technical Background

Open Cowork is built on the Kimi CLI agent architecture, which provides:
- **Multi-provider LLM support**: Works with OpenAI, Anthropic, Google, and other providers
- **Powerful tooling system**: File operations, shell commands, web scraping, and MCP integration
- **Subagent architecture**: Parallel task execution with isolated contexts
- **Context management**: Automatic compaction and session persistence

For technical details about the underlying agent system, see the [original Kimi CLI documentation](https://moonshotai.github.io/kimi-cli/en/).

## Development

### Open Cowork Desktop Development

> [!NOTE]
> Development setup for Open Cowork Windows Desktop App is in progress.

**Technology Stack (Planned):**
- **Backend**: Kimi CLI agent architecture (Python)
- **Frontend**: Modern Windows UI framework (evaluating WPF/WinUI 3/Electron)
- **IPC**: Communication layer between GUI and agent backend
- **Packaging**: Standalone Windows installer with embedded Python runtime

**Development Roadmap:**
1. ✅ Fork Kimi CLI codebase
2. 🔄 Design Windows desktop UI mockups
3. 📋 Implement GUI framework integration
4. 📋 Build IPC bridge between UI and Python backend
5. 📋 Package as standalone Windows installer
6. 📋 Add Windows-specific features (system tray, file explorer integration)

### CLI Backend Development

The CLI backend is based on Kimi CLI. To develop or test the CLI:

```sh
git clone https://github.com/watergold20222022/open-cowork.git
cd open-cowork

# Install uv if not already installed
curl -LsSf https://astral.sh/uv/install.sh | sh

# Prepare development environment (requires make)
make prepare

# Or manually with uv
uv sync --frozen --all-extras --all-packages
```

Then you can work with the CLI backend:

```sh
uv run kimi  # run the CLI agent

make format  # format code (or: uv run ruff format)
make check   # run linting and type checking
make test    # run tests
make build   # build python packages
```

## Contributing

We welcome contributions to Open Cowork! This project aims to bring AI-assisted coding to Windows desktop users.

**Areas where we need help:**
- Windows UI/UX design
- Desktop application development (WPF/WinUI/Electron)
- Python-to-GUI IPC integration
- Windows installer packaging
- Testing on various Windows versions

Please open an issue to discuss your ideas before submitting a pull request.

## Credits

This project is forked from [Kimi CLI](https://github.com/MoonshotAI/kimi-cli) by Moonshot AI. We're grateful for their excellent work on the agent architecture and tooling system.

**Original Kimi CLI:**
- Repository: https://github.com/MoonshotAI/kimi-cli
- Documentation: https://moonshotai.github.io/kimi-cli/en/
- License: See [LICENSE](./LICENSE) file

## License

This project maintains the same license as the original Kimi CLI. See [LICENSE](./LICENSE) for details.
