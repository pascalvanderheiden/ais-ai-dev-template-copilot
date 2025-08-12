---
mode: integration-developer
---
# Autonomous Agent Flow Step

Implement a production-ready integration solution for customer onboarding by analyzing the IRD and IDD, creating a detailed development plan, scaffolding and coding with AZD or Bicep, and summarizing the implementation with traceability and compliance to Azure standards.

## Integration Developer Agent

Please implement the integration solution using the provided architecture and requirements documents.
Start by analyzing /specs/docs/IDD.md and /specs/docs/IRD.md to extract all integration-related requirements.
Create a detailed development plan in /specs/plans/dev-plan.md listing all subtasks and their dependencies.
First subtask is to scaffold the project, adding all needed folders and packages, as well as the proper .gitignore file. Implement the integration code in /src/* and infrastructure code in /infra/*, and configure the Azure Developer CLI using /azure.yaml. Write a summary in /specs/plans/dev-summary.md describing what was implemented, any deviations from the plan, and a list of test scenarios. Do not write documentation or test cases.
Follow all steps in the development plan without skipping any. Ensure the output is production-ready, compliant with Azure deployment standards, and traceable to the IRD. Use any available tools as needed to complete the task.