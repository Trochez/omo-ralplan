# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-04-13

### Added
- Initial release of OMO-Ralplan skill
- Model-agnostic consensus planning workflow
- Planner → Architect → Critic review loop
- Sequential execution enforcement for consensus steps
- Pre-execution gate for underspecified requests
- Graceful degradation when subagents unavailable
- Configurable timeouts via environment variables
- Interactive and deliberate mode flags
- Comprehensive documentation and examples

### Features
- Native `task()` tool delegation (no MCP dependencies)
- Works with any configured AI model
- Environment variable-based timeout configuration
- Fallback to local analysis when needed
- Pre-context intake for grounded planning
- ADR (Architecture Decision Record) output format

### Documentation
- Complete README with installation instructions
- Skill datasheet with all arguments
- Usage examples for all modes
- Architecture diagrams
- Troubleshooting guide
- Configuration guide (`docs/configuration.md`)
- Session learnings document (`docs/learnings.md`)
- Example files for all usage patterns

### Design Decisions
- Use native `task()` instead of MCP tools for portability
- Sequential consensus workflow (Architect before Critic)
- Environment variable timeouts for configurability
- Graceful degradation for resilience
- Pre-execution gate to prevent wasted cycles
