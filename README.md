# FNF Prompt Library
A centralized, prompt library managed by FNFI Enterprise Architecture team.

## What is FNF Prompt Library?
FNF Prompt Library serves as the governed source for prompts, enabling standardization, reuse, and compliance across development teams. It provides tested prompts that align to best practices, helping teams deliver consistent results.

## What's Included
This repository contains:
- **Prompts** - Ready-to-use prompts for various development scenarios
- **Evaluation artifacts** - Test results and performance metrics
- **Documentation** - Usage guidelines, best practices
- **Approval records** - Governance and review history

**Not included:** Secrets, API keys, credentials, PII, or sensitive customer data.

## Featured Collections

### Services
Backend development prompts for testing and code quality:

- **[.NET Core Unit Testing](services/DotNETCore_UnitTesting_prompt.md)** - Generate comprehensive unit tests for .NET 8+ applications using xUnit, NSubstitute, and FluentAssertions. Includes results and detailed optimization report.

### UI
Frontend development prompts for modern web frameworks:

- **[Angular Unit Testing](ui/Angular_UnitTesting_Prompt.md)** - Generate Jasmine/Karma tests for Angular components and services.
- **[React Unit Testing](ui/React_UnitTesting_Prompt.md)** - Create Jest and React Testing Library test suites.
- **[Blazor Unit Testing](ui/Blazor_UnitTesting_Prompt.md)** - Build bUnit tests for Blazor components.

Each collection includes prompt templates

## Contributing
We welcome contributions from all teams! Please review our [Contribution Guidelines](Contribution.md) before submitting prompts or improvements.

## Code of Conduct
This project adheres to our [Code of Conduct](CodeOfConduct.md). By participating, you agree to maintain a respectful and inclusive environment.

## Review Process
:::mermaid
flowchart LR
    A[Contributor creates prompt] --> B[Add file to prompts/ directory]
    B --> C[Ensure formating & naming conventions]
    C --> D[Manual testing in Copilot / IDE]
    D --> E[Open Pull Request to main branch]
    E --> F[Maintainer review starts]
    F --> G{Meets contribution guidelines?}
    G -->|No| H[Review comments / change requests]
    H --> I[Contributor updates PR]
    I --> F
    G -->|Yes| J{Prompt quality OK?}
    J -->|No| H
    J -->|Yes| K[Approve Pull Request]
    K --> L[Merge into main branch]
    L --> M[Prompt available to all users]
:::
## Support
For questions, issues, or suggestions, please open an issue or contact the Enterprise Architecture team.
---
*Building better AI solutions, one prompt at a time.*

