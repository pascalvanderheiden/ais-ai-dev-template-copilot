# GitHub Copilot Instructions for Azure Integration Flow

## Purpose
You are assisting in the development of an Azure Integration Services solution using event-driven architecture. Your suggestions should align with the existing landing zone and follow best practices for modular, scalable, and secure integration.

## Technical Stack
- Reference Landing Zone: [azd-ais-lza](https://github.com/pascalvanderheiden/azd-ais-lza)
- Core services: API Management, Logic Apps, Azure Functions, Service Bus, Event Grid
- Authentication: Azure Managed Identity
- Infrastructure as Code: Bicep
- CI/CD: GitHub Actions
- Coding language: C#
- Event Model: CloudEvents v1.0+ specification
- Network: Private Endpoints and VNet Integration

## Documentation Guidelines
- Use Mermaid syntax for diagrams, exclude `()` in node names

## Developer Guidelines

### General
- Follow best practices for coding standards, including naming conventions and folder structure.
- Use this folder structure (`infra/`, `src/`, `tests/`)
- Generate infrastructure as code using Bicep modules under `infra/modules/`, for each module used create a seperate *.bicep file, referenced in `main.bicep` file.
- Design integrations using CloudEvents with required attributes: `id`, `source`, `specversion`, `type`
- Suggest code and configuration that supports loose coupling, idempotency, and correlation ID tracking
- Use managed identity for service-to-service authentication
- Store secrets in Key Vault and configure private endpoints where applicable
- Generate OpenAPI specs for APIs exposed via APIM
- Use GitHub Actions for CI/CD pipelines, including build and deployment stages
- Follow the Azure Well-Architected Framework principles
- Ensure all code is modular, reusable, and follows SOLID principles
- Prioritize Logic Apps Standard for workflows, before considering Azure Functions

### Bicep Infrastructure Patterns
- `${abbrs.resourceType}${resourceToken}` for unique naming of resources
- Use `existing` keyword to reference resources in the landing zone
- Integration Pattern deploys to new RG but utilzes resources from the discovered Azure Integration Services Landing Zone
- Each module in `infra/modules/` with parameters for customization
- Creates and links private DNS zones for private endpoints
- Outputs resource IDs and connection strings for use in application code
- Include tagging for cost management and governance
- Retrieve and use the current IP address for local deployment restrictions (Landing Zone already has a rule for the current IP address)

### Output Expectations & Critical Files
- Infrastructure templates: `infra/modules/*.bicep`
- Application code: `src/functions/`, `src/workflow/`, `src/apim/`
- Event schemas: `src/eventgrid/`
- Scripts: `/scripts/*.ps1`
- `/infra/main.bicep`: Main infrastructure template
- `/workflow/*/workflow.json`: Logic App workflow definitions
- `/infra/modules/`*: Bicep modules for reusable infrastructure components
- `/tests/`: Automated tests for integration scenarios
- `.github/workflows/`: CI/CD pipeline definitions

### Security Model
- Managed identities for all service-to-service communication
- Private endpoints for all connectivity, where applicable
- Key Vault for secrets management with role-based access
- API Management handles external authentication and rate limiting

## Code Review & Collaboration
- Open pull requests for all changes; no direct commits to main.
- Request at least one review before merging.
- Address all review comments before merging.
- Keep PRs focused and small.

## What Not to Do
- Do not suggest monolithic designs or synchronous coupling
- Do not hardcode secrets or credentials