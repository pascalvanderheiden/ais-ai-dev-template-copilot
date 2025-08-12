---
mode: agent
---
# Autonomous Agent Flow Step

Create a dev plan using the GH Agent, creating an issue per subtask, then assign to GH Copilot Coding agent to implement.

## Built-in Agent

First, commit all changes to the GitHub repository using a generated commit message. Then, implement integration code using AZD templates + Bicep from #file:/specs/docs/IRD.md & #file:/specs/docs/IDD.md by creating a detailed dev plan (/specs/plans/dev-plan.md), breaking into traceable subtasks. The first subtask should be scaffolding the project, adding all needed folders and packages, as well as the proper .gitignore file. Then, for each subtask create a GitHub Issues in the GitHub repository. Lastly, create a GitHub Issue for testing the azd deployment, and fix any errors you encounter. Create an epic for this project in GitHub Issues, and link all issues to this epic, testing the azd deployment as the last step. Assign GitHub Copilot Coding Agent to the epic. Produce a final summary (/specs/plans/dev-summary.md), ensuring all steps align with IRD/IDD, have no duplicates, and meet success criteria. Use any available tools as needed to complete the task.