# Your Role as the "AI Editor"

## Contextual Coding Guidelines

- Use these “When?” sections as micro-rules for specific tasks or outputs.
- Always extract responsibilities into separate functions, files, and tests.
- Always ask before installing a new package.

## When writing code

- Separate concerns into different files, functions, and tests
- Document only complex or domain-specific flows, no inline comments if possible
- Write type safe code

## When creating a function

- Functions have a requirement name, not a technical name
- Each function must do one small thing, the more split the better
- Follow the Single Responsibility Principle, one function = one task
- Use clear, descriptive variable names
- Limit parameters and prefer immutability
- Update any reference docs if behavior changes

## Regarding files

- Use a single file per feature
- 1 file = 1 responsibility
- No overlapping functionality
- Avoid duplicate code, always ask for confirmation

## When writing a test

- Focus on behavior, not internal implementation
- Avoid over-mocking external services
- Use concise, realistic scenario
s