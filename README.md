# Claude Code Plugins

A curated collection of Claude Code plugins for enhanced AI assistance. This marketplace provides high-quality plugins to extend Claude's capabilities with specialized expertise.

## Quick Start

### Add the Marketplace

```bash
/plugin marketplace add aazbeltran/claude-code-plugins
```

### Install Plugins

```bash
# Browse available plugins interactively
/plugin

# Or install directly
/plugin install testing@lxdb-plugins
```

## Available Plugins

### Testing

Expert guidance on software testing strategies, methodologies, and best practices.

**Features:**
- Language-agnostic testing advice (Python, JavaScript, Java, C#, Go, Rust, etc.)
- Test strategy design (unit, integration, E2E, contract testing)
- Methodology guidance (TDD, BDD, test-later approaches)
- CI/CD pipeline optimization
- Legacy code testing techniques
- Coverage strategy recommendations

**Use when:**
- Designing test strategies for new projects
- Choosing between testing approaches (TDD vs test-later)
- Refactoring existing test suites
- Optimizing test performance and reliability
- Testing legacy codebases

**Install:**
```bash
/plugin install testing@lxdb-plugins
```

**Usage:**
After installation, Claude will have access to comprehensive testing expertise. Simply ask questions about testing strategies, methodologies, or specific scenarios.

**Examples:**
- "How should I structure tests for a microservices architecture?"
- "What's the best approach for testing this legacy codebase?"
- "Should I use TDD or write tests after for this feature?"
- "How do I optimize my CI/CD pipeline's test execution?"

## Plugin Structure

Each plugin in this marketplace follows best practices:

```
plugins/
└── plugin-name/
    ├── plugin.json          # Plugin manifest
    ├── SKILL.md            # Main agent definition
    └── references/         # Supporting documentation
        └── *.md           # Detailed reference materials
```

## For Plugin Developers

### Adding Your Plugin

1. Create your plugin in the `plugins/` directory following the structure above
2. Add an entry to `.claude-plugin/marketplace.json`
3. Submit a pull request with:
   - Plugin code and documentation
   - Clear description and use cases
   - Testing evidence

### Plugin Requirements

- Must include `plugin.json` manifest
- Clear, concise description
- Well-documented use cases
- High-quality content
- MIT license (or compatible)

## Marketplace Management

### Update Marketplace

```bash
/plugin marketplace update lxdb-plugins
```

### List Installed Plugins

```bash
/plugin list
```

### Remove Marketplace

```bash
/plugin marketplace remove lxdb-plugins
```

## Contributing

Contributions are welcome! To contribute:

1. Fork this repository
2. Create a feature branch
3. Add or improve plugins
4. Test your changes locally
5. Submit a pull request

### Testing Locally

```bash
# Add local marketplace for testing
/plugin marketplace add ./path/to/claude-code-plugins

# Install plugin from local marketplace
/plugin install plugin-name@lxdb-plugins
```

## License

MIT License - See individual plugins for their specific licenses.

## Support

- **Issues**: [GitHub Issues](https://github.com/aazbeltran/claude-code-plugins/issues)
- **Discussions**: [GitHub Discussions](https://github.com/aazbeltran/claude-code-plugins/discussions)

## Version History

### 1.0.0 (Current)
- Initial marketplace release
- Testing plugin v1.0.0

## Acknowledgments

Built for the Claude Code community to enhance AI-assisted development workflows.
