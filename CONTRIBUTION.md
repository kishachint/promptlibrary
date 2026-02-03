# Contributing to FNF Prompt Library

Thank you for your interest in contributing to FNF Prompt Library! We welcome contributions from all teams to help expand our enterprise prompt library and improve AI development practices across the organization.

## How to Contribute

### Adding or Updating the Prompts

1. **Access the repository**: Ensure you have access to the FNF Prompt Library repository in Azure DevOps
2. **Create a feature branch**: Branch from `main` using a descriptive name (e.g., `prompt/dotnet-integration-testing` or `prompt/angular-accessibility`)
3. **Choose the right category**: Place your prompt in the appropriate directory:
   - `services/` - Backend, API, and service-layer prompts
   - `ui/` - Frontend framework and UI-related prompts
4. **Follow naming conventions**: Use descriptive, consistent filenames (e.g., `python_unittesting_prompt.md`)
5. **Structure your content**:
   - Start with a clear heading describing the prompt's purpose
   - Include use case, intended audience, and supported models
   - Provide specific, actionable instructions
   - Add examples where helpful
   - Document expected outputs and known limitations
6. **Test your prompt**: Validate the prompt against real scenarios and document results in a separate `*_results.md` file or link to the results file in *_results.md file.

### Documentation Standards

- **Be specific**: Generic prompts are less helpful than targeted, actionable guidance
- **Use clear language**: Write concisely and avoid jargon where possible
- **Include metadata**: Add version, owner, supported AI models, and last updated date
- **Provide context**: Explain when to use the prompt and what problem it solves
- **Add examples**: Show real-world usage and expected outcomes

## Submitting Your Contribution

1. **Raise a Pull Request**: Create a PR in Azure DevOps targeting the `main` branch
2. **Use the PR template below** for your description
3. **Address review feedback**: Respond to comments and make necessary updates
4. **Approval and merge**: Once approved, your contribution will be merged and versioned

### Pull Request Template

``` markdown
### Summary

Short overview of what your PR adds or changes.

### Details

- What files were added/changed:
  - `services/your-prompt.md`
  - `services/your-prompt-results.md`

- Why these additions/changes are useful:
  - Problem being solved
  - Target audience and use cases
  - Expected impact

### Tested

Describe how you tested the prompt (e.g., in VS Code with GitHub Copilot, Claude, or other AI tools):
- Test scenarios covered
- Sample inputs and outputs
- Edge cases validated

### Dependencies

List any prerequisites or related prompts (if applicable).
```

## What We Accept

We welcome prompts covering any development technology or practice that helps teams deliver better AI-powered solutions, ensure you limit to what is being used in the enterprise.

- Programming languages and frameworks (backend and frontend)
- Testing strategies (unit, integration, E2E)
- Code quality and best practices
- Architecture patterns and design principles
- Accessibility and inclusive design
- Performance optimization

## What We Don't Accept

To maintain a responsible and effective library:

- **Security violations**: Prompts that bypass security policies or expose vulnerabilities
- **Harmful content**: Guidance promoting malicious activities or discrimination
- **Policy circumvention**: Attempts to violate platform terms of service
- **Unvalidated prompts**: Contributions without testing evidence
- **Sensitive data**: PII, credentials, API keys, or proprietary information

## Quality Guidelines

- **Test thoroughly**: Validate against multiple scenarios and edge cases
- **Follow folder structure**: Maintain consistent organization
- **Write clear instructions**: Use bullet points and structured formatting
- **Promote best practices**: Encourage secure, maintainable, and ethical development
- **Keep it focused**: Each prompt should address a specific use case
- **Document limitations**: Be transparent about known constraints

## Code of Conduct

This project adheres to our [Code of Conduct](CodeOfConduct.md). By participating, you agree to maintain a respectful, inclusive, and collaborative environment.

---

*Every contribution helps build better AI solutions. Thank you for being part of FNF Prompt Library!*
