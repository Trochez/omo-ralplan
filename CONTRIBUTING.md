# Contributing to OMO-Ralplan

Thank you for your interest in contributing to OMO-Ralplan!

## How to Contribute

### Reporting Issues

1. Check if the issue has already been reported
2. Use the issue templates when creating new issues
3. Provide as much detail as possible:
   - Oh-My-OpenCode version
   - Configuration details
   - Steps to reproduce
   - Expected vs actual behavior

### Submitting Changes

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Test thoroughly
5. Commit your changes (`git commit -m 'Add amazing feature'`)
6. Push to the branch (`git push origin feature/amazing-feature`)
7. Open a Pull Request

### Development Guidelines

#### Code Style

- Follow existing patterns in the skill files
- Keep documentation up to date
- Use clear, descriptive commit messages

#### Testing

Before submitting a PR, verify:

- [ ] No `ask_codex` or MCP tool references
- [ ] No hardcoded model names
- [ ] All agent calls use `task()` tool
- [ ] All `task()` calls include `load_skills=[]`
- [ ] All `task()` calls include `run_in_background` parameter
- [ ] Architect and Critic calls are sequential
- [ ] Timeout configuration uses environment variables
- [ ] Fallback strategy documented

#### Documentation

- Update README.md if adding new features
- Update CHANGELOG.md with your changes
- Add examples for new functionality

### Questions?

Open a discussion or issue if you have questions about contributing.
