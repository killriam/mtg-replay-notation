# Contributing to MTG Replay Notation

Thank you for your interest in contributing! This document provides guidelines for contributing to the MTG Replay Notation specification.

## Ways to Contribute

### 1. Report Issues
- **Specification bugs**: Inconsistencies or errors in the spec
- **Missing features**: Functionality gaps for your use case
- **Documentation improvements**: Clarifications or examples needed

### 2. Propose Changes
- Open an issue describing the proposed change
- Discuss with maintainers before implementing major changes
- Submit a pull request with your changes

### 3. Share Implementations
- Link to your implementation in an issue
- We may add it to the README

## Specification Change Process

### Minor Changes (Patch Version)
- Typo fixes
- Documentation clarifications
- Additional examples

### Backward-Compatible Additions (Minor Version)
- New optional fields
- New event types
- New enumeration values

### Breaking Changes (Major Version)
- Changing required fields
- Removing fields
- Changing field types
- Semantic changes to existing fields

## Pull Request Guidelines

1. **One change per PR**: Keep PRs focused
2. **Update version**: If changing the spec, update version numbers
3. **Update CHANGELOG**: Document your changes
4. **Update schema**: Keep JSON schema in sync
5. **Add examples**: Include examples for new features

## Coding Style

### Markdown
- Use ATX-style headers (`#`, `##`, etc.)
- Use fenced code blocks with language specifiers
- One sentence per line for easier diffs

### JSON Examples
- Use 4-space indentation
- Use double quotes for strings
- Include comments via JSON5 or separate explanation

### Field Naming
- Use `snake_case` for JSON field names
- Use clear, descriptive names
- Prefer full words over abbreviations

## Testing Changes

Before submitting:
1. Validate all JSON examples against the schema
2. Check markdown renders correctly
3. Verify cross-references are accurate

## Code of Conduct

- Be respectful and constructive
- Focus on the technical merits
- Welcome newcomers

## Questions?

Open an issue with the "question" label.
