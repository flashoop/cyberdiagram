# PentestGPT Documentation

Comprehensive developer documentation for the PentestGPT project.

## Documentation Index

### 📂 [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md)
**Overview of project organization and file layout**

Learn about:
- Directory structure and file organization
- Core modules and their responsibilities
- Dependencies and build system
- Configuration files
- Test organization
- Runtime file locations

**When to read:** First time exploring the codebase, understanding project layout

---

### ✨ [FEATURES.md](./FEATURES.md)
**Comprehensive feature documentation**

Covers:
- Core features (Autonomous CTF solving, Intelligent methodology, Fallback strategies)
- Agent capabilities (Tool execution, Flag detection, Claude Code integration)
- User interface (TUI, Keyboard controls, Output modes)
- Session management (Persistence, Resume functionality)
- Benchmark system (Management, Structure, Running benchmarks)
- Telemetry & Observability
- Configuration & Authentication
- Advanced features

**When to read:** Understanding what PentestGPT can do, learning to use features

---

### 🏛️ [ARCHITECTURE.md](./ARCHITECTURE.md)
**Design patterns, architectural decisions, and system design**

Explores:
- Architectural overview and high-level design
- Design patterns (Singleton, Observer, Abstract Factory, Strategy, State, Command)
- Core components (AgentController, PentestAgent, EventBus, SessionStore)
- Data flow and message routing
- Event-driven architecture
- Session lifecycle
- Backend abstraction
- Extension points
- Performance considerations

**When to read:** Contributing code, understanding design decisions, extending the system

---

### 💻 [CODE_GUIDE.md](./CODE_GUIDE.md)
**Detailed code walkthroughs and implementation details**

Explains:
- Entry point & CLI implementation
- Agent Controller code walkthrough
- Pentest Agent execution flow
- Event Bus system implementation
- Backend abstraction layer
- Session management code
- TUI application structure
- Activity tracking system
- Telemetry integration
- Tool framework
- Configuration management

**When to read:** Understanding implementation details, debugging, making code changes

---

## Quick Navigation

### For New Contributors

1. Start with [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md) to understand the codebase layout
2. Read [ARCHITECTURE.md](./ARCHITECTURE.md) to learn the design patterns
3. Dive into [CODE_GUIDE.md](./CODE_GUIDE.md) for implementation details
4. Check [FEATURES.md](./FEATURES.md) for feature documentation

### For Users

1. Read [FEATURES.md](./FEATURES.md) to learn what PentestGPT can do
2. Reference [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md) for configuration file locations
3. Check the main [../README.md](../README.md) for installation and usage

### For Maintainers

1. Review [ARCHITECTURE.md](./ARCHITECTURE.md) for design decisions
2. Check [CODE_GUIDE.md](./CODE_GUIDE.md) for critical code paths
3. Reference [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md) for test organization

---

## Documentation Structure

```
docs/
├── README.md              # This file - documentation index
├── PROJECT_STRUCTURE.md   # Project organization and layout
├── FEATURES.md            # Feature documentation
├── ARCHITECTURE.md        # Design patterns and architecture
└── CODE_GUIDE.md          # Code walkthroughs and explanations
```

---

## Related Documentation

- **[../CLAUDE.md](../CLAUDE.md)** - Instructions for Claude Code AI assistant
- **[../README.md](../README.md)** - User-facing documentation and quick start
- **Code Comments** - Inline documentation in source files

---

## Contributing to Documentation

When updating the codebase, please also update relevant documentation:

1. **New Feature?** → Update `FEATURES.md`
2. **Architectural Change?** → Update `ARCHITECTURE.md`
3. **New Module?** → Update `CODE_GUIDE.md` and `PROJECT_STRUCTURE.md`
4. **Configuration Change?** → Update `PROJECT_STRUCTURE.md`

---

## Documentation Standards

### File Format
- All documentation in Markdown format
- Use GitHub Flavored Markdown (GFM)
- Include table of contents for long documents

### Code Examples
- Use syntax highlighting (```python, ```bash)
- Include comments for complex code
- Show both usage and implementation examples

### Structure
- Clear headings and sections
- Cross-references between documents
- "When to read" section for each document

### Maintenance
- Keep in sync with code changes
- Update examples when APIs change
- Add new sections as features are added

---

## Need Help?

- **Issues:** [GitHub Issues](https://github.com/GreyDGL/PentestGPT/issues)
- **Discord:** [Join Discord](https://discord.gg/eC34CEfEkK)
- **Paper:** [USENIX Security 2024](https://www.usenix.org/conference/usenixsecurity24/presentation/deng)

---

## License

This documentation is part of the PentestGPT project and is licensed under the MIT License.

See [../LICENSE.md](../LICENSE.md) for details.
