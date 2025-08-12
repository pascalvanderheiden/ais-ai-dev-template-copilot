# GitHub Copilot Instructions for Azure Integration Flow

## Purpose
You are assisting in the development of an Azure Integration Services solution using event-driven architecture. Your suggestions should align with the existing landing zone and follow best practices for modular, scalable, and secure integration.

## Technical Stack
- Reference Landing Zone: [azd-ais-lza](https://github.com/pascalvanderheiden/azd-ais-lza)
- Structure: [Azure Developer CLI (`azd`) templates](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/azd-templates)
- Event model: [CloudEvents v1.0+ specification](https://github.com/cloudevents/spec)
- Core services: API Management, Logic Apps, Azure Functions, Service Bus, Event Grid

## Developer Guidelines

### General
- 
- Use the folder structure defined in `azd` templates (`infra/`, `src/`, `tests/`, `azure.yaml`)
- Generate infrastructure as code using Bicep modules under `infra/modules/`, for each module used create a seperate *.bicep file, referenced in `main.bicep` file.
- Design integrations using CloudEvents with required attributes: `id`, `source`, `specversion`, `type`
- Suggest code and configuration that supports loose coupling, idempotency, and correlation ID tracking
- Use managed identity for service-to-service authentication
- Store secrets in Key Vault and configure private endpoints where applicable
- Generate OpenAPI specs for APIs exposed via APIM
- Use GitHub Actions for CI/CD aligned with `azd` templates
- Follow the Azure Well-Architected Framework principles
- Ensure all code is modular, reusable, and follows SOLID principles
- Prioritize Logic Apps Standard for workflows, before considering Azure Functions

### AZD Deployment Scripts
1. **preprovision.ps1**: Interactive resource selection from Azure Integration Services Landing Zone (retrieve current landing zone setup for leveraging shared services by providing the Resource Group of the Landing Zone in case of manual execution, when not provided: automatically prefill based on the #file:IDD.md)
2. **predeploy.ps1**: Pre-deployment configuration
3. **postdeploy.ps1**: Post-deployment setup and API Management configuration

### Bicep Infrastructure Patterns
- **Resource Token**: `${abbrs.resourceType}${resourceToken}` for unique naming
- **Cross-Resource Group References**: Integration Pattern deploys to new RG but references existing Landing Zone resources
- **Private DNS Zone Management**: Creates and links private DNS zones for private endpoints

### Logic Apps Structure
- Located in `/workflow/` directory
- Each workflow has its own folder with `workflow.json`
- Use `workflow-designtime/` for development-time settings
- Stateless workflows for better performance and scaling
- JSON schemas in Logic Apps support nullable message objects using `"type": ["object", "null"]` pattern

### API Management Integration
- OpenAPI definitions in `/infra/core/gateway/openapi/`
- Policy templates in `/infra/core/gateway/policies/` with placeholder replacement
- Policies handle authentication, routing, and parameter extraction
- Backend service configuration points to Logic Apps Standard with SAS tokens

### Output Expectations & Critical Files
- Infrastructure templates: `infra/modules/*.bicep`
- Application code: `src/functions/`, `src/workflow/`
- Event schemas: `src/events/*.json`
- Scripts: `/scripts/*.ps1`
- `/azure.yaml`: AZD configuration with PowerShell hooks
- `/infra/main.bicep`: Main infrastructure template with Landing Zone integration
- `/workflow/*/workflow.json`: Logic App workflow definitions
- `/scripts/preprovision.ps1`: Interactive Landing Zone resource discovery
- `/infra/core/gateway/policies/api-policy.xml`: API Management policy template

### Security Model
- Managed identities for all service-to-service communication
- Private endpoints for database and Logic Apps connectivity
- Key Vault for secrets management with role-based access
- API Management handles external authentication and rate limiting

### Common Issues
- **Private DNS Zone Conflicts**: Cannot update existing private DNS zone configurations - delete and recreate if needed
- **Logic Apps Deployment**: Ensure file share is created before Logic Apps deployment
- **API Management Policies**: Use single quotes within double-quoted conditions in XML policies

## Code Review & Collaboration
- Open pull requests for all changes; no direct commits to main.
- Request at least one review before merging.
- Address all review comments before merging.
- Keep PRs focused and small.

## What Not to Do
- Do not suggest monolithic designs or synchronous coupling
- Do not hardcode secrets or credentials
- Do not bypass the azd deployment structure