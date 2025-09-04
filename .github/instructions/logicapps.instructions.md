---
description: 'Best practices for developing, securing, monitoring, and deploying Azure Logic Apps workflows to existing App Service plans'
applyTo: "**/src/workflow/**/*,**/infra/**/*"
---

# Azure Logic Apps Development and Deployment Best Practices

## Overview

This guide provides comprehensive best practices for developing, securing, monitoring, and deploying **Azure Logic Apps (Standard)** workflows to **existing App Service plans** using GitHub Actions and Infrastructure as Code (Bicep). Focus is on managed identity authentication, enterprise integration patterns, and automated deployment for scalable workflow automation.

## Prerequisites

- Existing Azure App Service plan (for Logic Apps Standard)
- GitHub repository with Actions enabled
- Azure service principal or workload identity for deployment
- Bicep CLI and Azure CLI installed for local development
- Understanding of Workflow Definition Language (WDL) and integration patterns

## Project Structure

```
src/
├── workflow/
│   ├── workflows/
│   │   ├── [workflow-name]/
│   │   │   ├── workflow.json
│   │   │   ├── parameters.json
│   │   │   └── connections.json
│   ├── lib/
│   │   ├── schemas/
│   │   │   └── [schema-name].json
│   │   ├── templates/
│   │   │   └── [template-name].json
│   │   └── maps/
│   │       └── [map-name].liquid
│   ├── host.json
│   ├── local.settings.json
│   └── requirements.psd1
infra/
├── modules/
│   ├── logicapp/
│   │   ├── logicapp.bicep
│   │   ├── workflows.bicep
│   │   ├── connections.bicep
│   │   └── monitoring.bicep
│   └── networking/
│       └── private-endpoints.bicep
└── main.bicep
```

## Development Guidelines
- Stateless workflows for better performance and scaling
- JSON schemas in Logic Apps support nullable message objects using `"type": ["object", "null"]` pattern
- Use managed identity for all service-to-service authentication
- Store sensitive data in Azure Key Vault, accessed via managed identity
- Always use API Management for exposing Logic Apps to external consumers
- Create a API Management backend for each Logic App, handling authentication and rate limiting
- Ensure file share is created on Azure Storagebefore Logic Apps deployment

## Security Best Practices

### 1. Managed Identity Authentication (Recommended)

#### Connection Configuration with Managed Identity
```json
{
  "connections": {
    "serviceBus": {
      "connectionRuntimeUrl": "https://servicebus.azure.com",
      "authentication": {
        "type": "ManagedServiceIdentity"
      }
    },
    "keyVault": {
      "connectionRuntimeUrl": "https://vault.azure.net",
      "authentication": {
        "type": "ManagedServiceIdentity"
      }
    }
  }
}
```

#### Workflow with Managed Identity Authentication
```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "actions": {
      "Get_Secret_From_KeyVault": {
        "type": "Http",
        "inputs": {
          "method": "GET",
          "uri": "https://@{parameters('keyVaultName')}.vault.azure.net/secrets/@{parameters('secretName')}?api-version=2016-10-01",
          "authentication": {
            "type": "ManagedServiceIdentity",
            "audience": "https://vault.azure.net"
          }
        }
      },
      "Call_Secure_API": {
        "type": "Http",
        "inputs": {
          "method": "POST",
          "uri": "@parameters('apiEndpoint')",
          "headers": {
            "Authorization": "Bearer @{body('Get_Secret_From_KeyVault')?['value']}",
            "Content-Type": "application/json"
          },
          "body": "@triggerBody()"
        },
        "runAfter": {
          "Get_Secret_From_KeyVault": ["Succeeded"]
        }
      }
    }
  }
}
```

### 2. Network Security and Private Endpoints

```json
// Private networking configuration for Logic Apps Standard
{
  "definition": {
    "actions": {
      "Call_Internal_API": {
        "type": "Http",
        "inputs": {
          "method": "GET",
          "uri": "https://internal-api.private.com/data",
          "headers": {
            "Content-Type": "application/json"
          }
        },
        "runtimeConfiguration": {
          "contentTransfer": {
            "transferMode": "Chunked"
          },
          "secureData": {
            "properties": ["inputs", "outputs"]
          }
        }
      }
    }
  }
}
```

### 3. Input Validation and Security

```json
{
  "definition": {
    "triggers": {
      "manual": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "customerId": {
                "type": "string",
                "pattern": "^[a-zA-Z0-9-]{1,50}$"
              },
              "amount": {
                "type": "number",
                "minimum": 0,
                "maximum": 100000
              }
            },
            "required": ["customerId", "amount"],
            "additionalProperties": false
          },
          "method": "POST"
        },
        "operationOptions": "EnableSchemaValidation"
      }
    },
    "actions": {
      "Validate_And_Sanitize_Input": {
        "type": "Compose",
        "inputs": {
          "customerId": "@{trim(triggerBody()?['customerId'])}",
          "amount": "@{float(triggerBody()?['amount'])}",
          "timestamp": "@{utcNow()}",
          "correlationId": "@{workflow().run.id}"
        }
      }
    }
  }
}
```

## Infrastructure as Code with Bicep

### 1. Main Logic App Module (infra/main.bicep)

```bicep
targetScope = 'resourceGroup'

@description('Name of the existing App Service plan')
param appServicePlanName string

@description('Name of the Logic App')
param logicAppName string

@description('Environment name (dev, staging, prod)')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

@description('Location for resources')
param location string = resourceGroup().location

@description('Application Insights connection string')
param applicationInsightsConnectionString string = ''

@description('Enable private endpoints')
param enablePrivateEndpoints bool = false

@description('VNet ID for private endpoints')
param vnetId string = ''

@description('Private endpoint subnet ID')
param privateEndpointSubnetId string = ''

// Reference existing App Service plan
resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' existing = {
  name: appServicePlanName
}

// Deploy Logic App Standard
module logicAppModule 'modules/logicapp/logicapp.bicep' = {
  name: 'logicapp-deployment'
  params: {
    logicAppName: logicAppName
    environment: environment
    location: location
    appServicePlanId: appServicePlan.id
    applicationInsightsConnectionString: applicationInsightsConnectionString
    enablePrivateEndpoints: enablePrivateEndpoints
    vnetId: vnetId
    privateEndpointSubnetId: privateEndpointSubnetId
  }
}

// Deploy monitoring
module monitoringModule 'modules/logicapp/monitoring.bicep' = {
  name: 'monitoring-deployment'
  params: {
    logicAppName: logicAppModule.outputs.logicAppName
    location: location
    environment: environment
  }
}

// Deploy connections
module connectionsModule 'modules/logicapp/connections.bicep' = {
  name: 'connections-deployment'
  params: {
    logicAppName: logicAppModule.outputs.logicAppName
    location: location
    environment: environment
    logicAppPrincipalId: logicAppModule.outputs.principalId
  }
}

// Outputs
output logicAppId string = logicAppModule.outputs.logicAppId
output logicAppName string = logicAppModule.outputs.logicAppName
output defaultHostName string = logicAppModule.outputs.defaultHostName
```

### 2. Logic App Module (infra/modules/logicapp/logicapp.bicep)

```bicep
@description('Logic App name')
param logicAppName string

@description('Environment name')
param environment string

@description('Location')
param location string

@description('App Service Plan resource ID')
param appServicePlanId string

@description('Application Insights connection string')
param applicationInsightsConnectionString string

@description('Enable private endpoints')
param enablePrivateEndpoints bool = false

@description('VNet ID for private endpoints')
param vnetId string = ''

@description('Private endpoint subnet ID')
param privateEndpointSubnetId string = ''

@description('Storage account name for workflows')
param storageAccountName string = '${replace(logicAppName, '-', '')}storage'

// Storage account for Logic App workflows
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: take(storageAccountName, 24)
  location: location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
  }
  properties: {
    accessTier: 'Hot'
    allowBlobPublicAccess: false
    allowSharedKeyAccess: false
    defaultToOAuthAuthentication: true
    minimumTlsVersion: 'TLS1_2'
    networkAcls: {
      bypass: 'AzureServices'
      defaultAction: enablePrivateEndpoints ? 'Deny' : 'Allow'
    }
    supportsHttpsTrafficOnly: true
    isHnsEnabled: false
  }
}

// Logic App (Standard)
resource logicApp 'Microsoft.Web/sites@2023-01-01' = {
  name: '${logicAppName}-${environment}'
  location: location
  kind: 'functionapp,workflowapp'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlanId
    httpsOnly: true
    clientAffinityEnabled: false
    publicNetworkAccess: enablePrivateEndpoints ? 'Disabled' : 'Enabled'
    vnetRouteAllEnabled: enablePrivateEndpoints
    siteConfig: {
      numberOfWorkers: 1
      linuxFxVersion: 'Node|18'
      alwaysOn: true
      ftpsState: 'Disabled'
      minTlsVersion: '1.2'
      http20Enabled: true
      functionAppScaleLimit: 0
      minimumElasticInstanceCount: 0
      appSettings: [
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${az.environment().suffixes.storage};AccountKey=${storageAccount.listKeys().keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTAZUREFILECONNECTIONSTRING'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${az.environment().suffixes.storage};AccountKey=${storageAccount.listKeys().keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTSHARE'
          value: toLower(logicAppName)
        }
        {
          name: 'AzureFunctionsJobHost__extensionBundle__id'
          value: 'Microsoft.Azure.Functions.ExtensionBundle.Workflows'
        }
        {
          name: 'AzureFunctionsJobHost__extensionBundle__version'
          value: '[1.*, 2.0.0)'
        }
        {
          name: 'APP_KIND'
          value: 'workflowApp'
        }
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: applicationInsightsConnectionString
        }
        {
          name: 'ApplicationInsightsAgent_EXTENSION_VERSION'
          value: '~3'
        }
        {
          name: 'XDT_MicrosoftApplicationInsights_Mode'
          value: 'Recommended'
        }
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: 'node'
        }
        {
          name: 'WEBSITE_NODE_DEFAULT_VERSION'
          value: '~18'
        }
        {
          name: 'WORKFLOWS_TENANT_ID'
          value: subscription().tenantId
        }
        {
          name: 'WORKFLOWS_SUBSCRIPTION_ID'
          value: subscription().subscriptionId
        }
        {
          name: 'WORKFLOWS_RESOURCE_GROUP_NAME'
          value: resourceGroup().name
        }
        {
          name: 'WORKFLOWS_LOCATION_NAME'
          value: location
        }
      ]
      connectionStrings: []
    }
  }
}

// Storage account role assignment for Logic App
resource storageAccountContributorRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storageAccount
  name: guid(storageAccount.id, logicApp.id, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe') // Storage Blob Data Contributor
    principalId: logicApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Private endpoint for Logic App (if enabled)
resource logicAppPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = if (enablePrivateEndpoints && !empty(vnetId)) {
  name: '${logicApp.name}-pe'
  location: location
  properties: {
    subnet: {
      id: privateEndpointSubnetId
    }
    privateLinkServiceConnections: [
      {
        name: 'logicAppConnection'
        properties: {
          privateLinkServiceId: logicApp.id
          groupIds: ['sites']
        }
      }
    ]
  }
}

// Outputs
output logicAppId string = logicApp.id
output logicAppName string = logicApp.name
output defaultHostName string = logicApp.properties.defaultHostName
output principalId string = logicApp.identity.principalId
output storageAccountName string = storageAccount.name
```

### 3. Connections Module (infra/modules/logicapp/connections.bicep)

```bicep
@description('Logic App name')
param logicAppName string

@description('Environment name')
param environment string

@description('Location')
param location string

@description('Logic App principal ID for RBAC')
param logicAppPrincipalId string

@description('Service Bus namespace name')
param serviceBusNamespaceName string = ''

@description('Key Vault name')
param keyVaultName string = ''

// Service Bus connection (if Service Bus namespace is provided)
resource serviceBusConnection 'Microsoft.Web/connections@2016-06-01' = if (!empty(serviceBusNamespaceName)) {
  name: 'servicebus-${environment}'
  location: location
  properties: {
    displayName: 'Service Bus Connection'
    api: {
      id: subscriptionResourceId('Microsoft.Web/locations/managedApis', location, 'servicebus')
    }
    parameterValueSet: {
      name: 'managedIdentityAuth'
      values: {
        namespaceEndpoint: {
          value: 'sb://${serviceBusNamespaceName}.servicebus.windows.net'
        }
      }
    }
  }
}

// Key Vault connection (if Key Vault name is provided)
resource keyVaultConnection 'Microsoft.Web/connections@2016-06-01' = if (!empty(keyVaultName)) {
  name: 'keyvault-${environment}'
  location: location
  properties: {
    displayName: 'Key Vault Connection'
    api: {
      id: subscriptionResourceId('Microsoft.Web/locations/managedApis', location, 'keyvault')
    }
    parameterValueSet: {
      name: 'managedIdentityAuth'
      values: {
        vaultName: {
          value: keyVaultName
        }
      }
    }
  }
}

// Office 365 Outlook connection
resource office365Connection 'Microsoft.Web/connections@2016-06-01' = {
  name: 'office365-${environment}'
  location: location
  properties: {
    displayName: 'Office 365 Outlook'
    api: {
      id: subscriptionResourceId('Microsoft.Web/locations/managedApis', location, 'office365')
    }
    parameterValueSet: {
      name: 'oauthMI'
      values: {}
    }
  }
}

// Application Insights connection
resource applicationInsightsConnection 'Microsoft.Web/connections@2016-06-01' = {
  name: 'applicationinsights-${environment}'
  location: location
  properties: {
    displayName: 'Application Insights'
    api: {
      id: subscriptionResourceId('Microsoft.Web/locations/managedApis', location, 'applicationinsights')
    }
    parameterValueSet: {
      name: 'managedIdentityAuth'
      values: {}
    }
  }
}

// RBAC assignments for Service Bus (if provided)
resource serviceBusDataSenderRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = if (!empty(serviceBusNamespaceName)) {
  scope: resourceGroup()
  name: guid(resourceGroup().id, logicAppPrincipalId, '69a216fc-b8fb-44d8-bc22-1f3c2cd27a39')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '69a216fc-b8fb-44d8-bc22-1f3c2cd27a39') // Service Bus Data Sender
    principalId: logicAppPrincipalId
    principalType: 'ServicePrincipal'
  }
}

resource serviceBusDataReceiverRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = if (!empty(serviceBusNamespaceName)) {
  scope: resourceGroup()
  name: guid(resourceGroup().id, logicAppPrincipalId, '4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0') // Service Bus Data Receiver
    principalId: logicAppPrincipalId
    principalType: 'ServicePrincipal'
  }
}

// RBAC assignments for Key Vault (if provided)
resource keyVaultSecretsUserRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = if (!empty(keyVaultName)) {
  scope: resourceGroup()
  name: guid(resourceGroup().id, logicAppPrincipalId, '4633458b-17de-408a-b874-0445c86b69e6')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6') // Key Vault Secrets User
    principalId: logicAppPrincipalId
    principalType: 'ServicePrincipal'
  }
}

// Outputs
output serviceBusConnectionId string = !empty(serviceBusNamespaceName) ? serviceBusConnection.id : ''
output keyVaultConnectionId string = !empty(keyVaultName) ? keyVaultConnection.id : ''
output office365ConnectionId string = office365Connection.id
output applicationInsightsConnectionId string = applicationInsightsConnection.id
```

## GitHub Actions CI/CD Pipeline

### Main Workflow (.github/workflows/logicapp-deploy.yml)

```yaml
name: Deploy Azure Logic Apps

on:
  push:
    branches: [main]
    paths: 
      - 'src/workflow/**'
      - 'infra/**'
  pull_request:
    branches: [main]
    paths:
      - 'src/workflow/**'
      - 'infra/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev
          - staging
          - prod

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  validate:
    name: Validate Logic App Configuration
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js for Logic Apps
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Validate workflow definitions
        run: |
          # Install Logic Apps CLI
          npm install -g @azure/logic-apps-cli
          
          # Validate workflow JSON files
          for workflow in src/workflow/workflows/*/workflow.json; do
            echo "Validating $workflow"
            if ! jq . "$workflow" > /dev/null 2>&1; then
              echo "❌ Invalid JSON in $workflow"
              exit 1
            fi
            echo "✅ Valid JSON: $workflow"
          done
          
          # Validate parameters files
          for params in src/workflow/workflows/*/parameters.json; do
            if [ -f "$params" ]; then
              echo "Validating $params"
              jq . "$params" > /dev/null || exit 1
              echo "✅ Valid parameters: $params"
            fi
          done

      - name: Setup Bicep
        uses: Azure/bicep-build-action@v1.0.1
        with:
          bicepFilePath: infra/main.bicep

      - name: Lint Bicep templates
        run: |
          az bicep build --file infra/main.bicep
          az bicep lint --file infra/main.bicep

  security-scan:
    name: Security and Compliance Scan
    runs-on: ubuntu-latest
    needs: validate
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Checkov security scan
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: bicep
          output_format: sarif
          output_file_path: reports/checkov-results.sarif

      - name: Upload Checkov results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: reports/checkov-results.sarif

      - name: Scan workflow definitions for secrets
        run: |
          # Check for hardcoded secrets in workflow definitions
          if grep -r -i -E "(password|key|token|secret)" src/workflow/workflows/ --include="*.json"; then
            echo "❌ Potential hardcoded secrets found in workflow definitions"
            exit 1
          else
            echo "✅ No hardcoded secrets detected"
          fi

  plan:
    name: Plan Infrastructure Changes
    runs-on: ubuntu-latest
    needs: [validate, security-scan]
    if: github.event_name == 'pull_request'
    environment: 
      name: dev
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Bicep What-If
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP }}
          template: infra/main.bicep
          parameters: >
            appServicePlanName=${{ vars.APP_SERVICE_PLAN_NAME }}
            logicAppName=${{ vars.LOGIC_APP_NAME }}
            environment=dev
            applicationInsightsConnectionString="${{ secrets.APPLICATION_INSIGHTS_CONNECTION_STRING }}"
          additionalArguments: --what-if --what-if-result-format FullResourcePayloads

      - name: Comment What-If Results
        uses: actions/github-script@v7
        with:
          script: |
            const output = `
            ## Logic Apps Configuration Changes Preview 🔄
            
            The infrastructure what-if analysis has been completed. Please review the changes above.
            
            - ✅ **Validation**: Workflow definitions and Bicep templates validated
            - ✅ **Security**: Checkov security scan completed  
            - 📋 **Changes**: Review the what-if output above
            
            **Resources to be modified:**
            - Check the what-if output for specific Logic App and connection changes
            - Review workflow deployment configurations
            - Validate App Service plan integration
            
            *This comment will be updated when the PR is merged and deployment completes.*
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: [validate, security-scan]
    if: github.ref == 'refs/heads/main'
    environment:
      name: dev
      url: https://portal.azure.com/#resource${{ vars.LOGIC_APP_RESOURCE_ID }}/overview
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Logic App Infrastructure
        uses: azure/arm-deploy@v1
        id: deploy-infra
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP }}
          template: infra/main.bicep
          parameters: >
            appServicePlanName=${{ vars.APP_SERVICE_PLAN_NAME }}
            logicAppName=${{ vars.LOGIC_APP_NAME }}
            environment=dev
            applicationInsightsConnectionString="${{ secrets.APPLICATION_INSIGHTS_CONNECTION_STRING }}"
            enablePrivateEndpoints=false
          deploymentName: logicapp-deployment-${{ github.run_number }}

      - name: Deploy Logic App Workflows
        run: |
          # Install Azure Functions Core Tools for Logic Apps deployment
          npm install -g azure-functions-core-tools@4 --unsafe-perm true
          
          # Deploy workflows to Logic App
          cd src/workflow
          
          # Create deployment package
          echo "Creating deployment package..."
          zip -r ../logicapp-package.zip . -x "local.settings.json"
          
          # Deploy using Azure CLI
          echo "Deploying Logic App workflows..."
          az logicapp deployment source config-zip \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --name ${{ vars.LOGIC_APP_NAME }}-dev \
            --src ../logicapp-package.zip \
            --timeout 600

      - name: Validate Logic App Deployment
        run: |
          echo "Validating Logic App deployment..."
          
          # Wait for deployment to complete
          sleep 30
          
          # Check Logic App status
          status=$(az logicapp show \
            --name ${{ vars.LOGIC_APP_NAME }}-dev \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --query "state" -o tsv)
          
          if [ "$status" = "Running" ]; then
            echo "✅ Logic App is running successfully"
          else
            echo "❌ Logic App deployment failed or not running. Status: $status"
            exit 1
          fi
          
          # List deployed workflows
          echo "Deployed workflows:"
          az logicapp workflow list \
            --name ${{ vars.LOGIC_APP_NAME }}-dev \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --output table

      - name: Test Workflow Endpoints (Optional)
        if: contains(github.event.head_commit.message, '[test-workflows]')
        run: |
          echo "Testing workflow endpoints..."
          
          # Get Logic App host name
          hostname=$(az logicapp show \
            --name ${{ vars.LOGIC_APP_NAME }}-dev \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --query "defaultHostName" -o tsv)
          
          echo "Logic App hostname: $hostname"
          
          # Test HTTP trigger workflows (if any)
          # Add specific tests based on your workflow configurations

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://portal.azure.com/#resource${{ vars.LOGIC_APP_RESOURCE_ID_STAGING }}/overview
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Staging
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP_STAGING }}
          template: infra/main.bicep
          parameters: >
            appServicePlanName=${{ vars.APP_SERVICE_PLAN_NAME_STAGING }}
            logicAppName=${{ vars.LOGIC_APP_NAME }}
            environment=staging
            applicationInsightsConnectionString="${{ secrets.APPLICATION_INSIGHTS_CONNECTION_STRING_STAGING }}"
            enablePrivateEndpoints=true
            vnetId=${{ vars.VNET_ID_STAGING }}
            privateEndpointSubnetId=${{ vars.PRIVATE_ENDPOINT_SUBNET_ID_STAGING }}
          deploymentName: logicapp-staging-${{ github.run_number }}

      - name: Deploy Staging Workflows
        run: |
          cd src/workflow
          zip -r ../logicapp-package.zip . -x "local.settings.json"
          
          az logicapp deployment source config-zip \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP_STAGING }} \
            --name ${{ vars.LOGIC_APP_NAME }}-staging \
            --src ../logicapp-package.zip \
            --timeout 600

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://portal.azure.com/#resource${{ vars.LOGIC_APP_RESOURCE_ID_PROD }}/overview
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Production
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP_PROD }}
          template: infra/main.bicep
          parameters: >
            appServicePlanName=${{ vars.APP_SERVICE_PLAN_NAME_PROD }}
            logicAppName=${{ vars.LOGIC_APP_NAME }}
            environment=prod
            applicationInsightsConnectionString="${{ secrets.APPLICATION_INSIGHTS_CONNECTION_STRING_PROD }}"
            enablePrivateEndpoints=true
            vnetId=${{ vars.VNET_ID_PROD }}
            privateEndpointSubnetId=${{ vars.PRIVATE_ENDPOINT_SUBNET_ID_PROD }}
          deploymentName: logicapp-prod-${{ github.run_number }}

      - name: Deploy Production Workflows
        run: |
          cd src/workflow
          zip -r ../logicapp-package.zip . -x "local.settings.json"
          
          az logicapp deployment source config-zip \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP_PROD }} \
            --name ${{ vars.LOGIC_APP_NAME }}-prod \
            --src ../logicapp-package.zip \
            --timeout 600

      - name: Post-deployment Validation
        run: |
          echo "🚀 Production deployment completed!"
          
          # Verify Logic App is running
          status=$(az logicapp show \
            --name ${{ vars.LOGIC_APP_NAME }}-prod \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP_PROD }} \
            --query "state" -o tsv)
          
          if [ "$status" = "Running" ]; then
            echo "✅ Production Logic App verified successfully"
          else
            echo "❌ Production Logic App not running. Status: $status"
            exit 1
          fi

  monitoring-setup:
    name: Configure Monitoring and Alerts
    runs-on: ubuntu-latest
    needs: [deploy-prod]
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Verify Monitoring Configuration
        run: |
          echo "Verifying monitoring and alerts setup..."
          
          # Check Application Insights configuration
          ai_key=$(az logicapp config appsettings list \
            --name ${{ vars.LOGIC_APP_NAME }}-prod \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP_PROD }} \
            --query "[?name=='APPLICATIONINSIGHTS_CONNECTION_STRING'].value | [0]" -o tsv)
          
          if [ ! -z "$ai_key" ]; then
            echo "✅ Application Insights configured"
          else
            echo "❌ Application Insights not configured"
          fi
          
          echo "✅ Monitoring configuration verified"

  notify:
    name: Notify Teams
    runs-on: ubuntu-latest
    needs: [deploy-prod, monitoring-setup]
    if: always()
    steps:
      - name: Notify on Success
        if: needs.deploy-prod.result == 'success'
        uses: 8398a7/action-slack@v3
        with:
          status: success
          fields: repo,commit,author,took
          text: "Logic Apps deployment successful! All workflows are deployed and running."
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      - name: Notify on Failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          fields: repo,commit,author,took
          text: "Logic Apps deployment failed! Please check the logs."
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## Best Practices for Workflow Development

### 1. Triggers and Actions Design

#### HTTP Trigger with Proper Schema Validation
```json
{
  "definition": {
    "triggers": {
      "manual": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "$schema": "http://json-schema.org/draft-07/schema#",
            "type": "object",
            "properties": {
              "customerId": {
                "type": "string",
                "pattern": "^[a-zA-Z0-9-]{1,50}$",
                "description": "Unique customer identifier"
              },
              "orderData": {
                "type": "object",
                "properties": {
                  "items": {
                    "type": "array",
                    "items": {
                      "type": "object",
                      "properties": {
                        "productId": {"type": "string"},
                        "quantity": {"type": "integer", "minimum": 1},
                        "price": {"type": "number", "minimum": 0}
                      },
                      "required": ["productId", "quantity", "price"]
                    }
                  },
                  "totalAmount": {
                    "type": "number",
                    "minimum": 0
                  }
                },
                "required": ["items", "totalAmount"]
              }
            },
            "required": ["customerId", "orderData"],
            "additionalProperties": false
          },
          "method": "POST"
        }
      }
    }
  }
}
```

#### Service Bus Trigger with Session Support
```json
{
  "definition": {
    "triggers": {
      "When_a_message_is_received": {
        "type": "ApiConnectionWebhook",
        "inputs": {
          "host": {
            "connection": {
              "referenceName": "servicebus"
            }
          },
          "body": {
            "properties": {
              "isSessionsEnabled": true,
              "subscriptionType": "Main"
            }
          },
          "path": "/subscriptions/@{encodeURIComponent('orders-topic')}/messages/batch/head",
          "queries": {
            "maxMessageCount": 10,
            "subscriptionName": "order-processing"
          }
        },
        "splitOn": "@triggerBody()"
      }
    }
  }
}
```

### 3. Error Handling and Reliability

- **Implement robust error handling**:
  - Use "runAfter" configurations to handle failures
  - Configure retry policies for transient errors
  - Use scopes with "runAfter" conditions for error branches
- **Implement fallback mechanisms** for critical operations
- **Add timeouts** for external service calls
- **Use runAfter conditions** for complex error handling scenarios

```json
"actions": {
  "HTTP_Action": {
    "type": "Http",
    "inputs": { },
    "retryPolicy": {
      "type": "fixed",
      "count": 3,
      "interval": "PT20S",
      "minimumInterval": "PT5S",
      "maximumInterval": "PT1H"
    }
  },
  "Handle_Success": {
    "type": "Scope",
    "actions": { },
    "runAfter": {
      "HTTP_Action": ["Succeeded"]
    }
  },
  "Handle_Failure": {
    "type": "Scope",
    "actions": {
      "Log_Error": {
        "type": "ApiConnection",
        "inputs": {
          "host": {
            "connection": {
              "name": "@parameters('$connections')['loganalytics']['connectionId']"
            }
          },
          "method": "post",
          "body": {
            "LogType": "WorkflowError",
            "ErrorDetails": "@{actions('HTTP_Action').outputs.body}",
            "StatusCode": "@{actions('HTTP_Action').outputs.statusCode}"
          }
        }
      },
      "Send_Notification": {
        "type": "ApiConnection",
        "inputs": {
          "host": {
            "connection": {
              "name": "@parameters('$connections')['office365']['connectionId']"
            }
          },
          "method": "post",
          "path": "/v2/Mail",
          "body": {
            "To": "support@contoso.com",
            "Subject": "Workflow Error - HTTP Call Failed",
            "Body": "<p>The HTTP call failed with status code: @{actions('HTTP_Action').outputs.statusCode}</p>"
          }
        },
        "runAfter": {
          "Log_Error": ["Succeeded"]
        }
      }
    },
    "runAfter": {
      "HTTP_Action": ["Failed", "TimedOut"]
    }
  }
}
```

### 4. Expressions and Functions

- **Use built-in expression functions** to transform data
- **Keep expressions concise and readable**
- **Document complex expressions** with comments

Common expression patterns:
- String manipulation: `concat()`, `replace()`, `substring()`
- Collection operations: `filter()`, `map()`, `select()`
- Conditional logic: `if()`, `and()`, `or()`, `equals()`
- Date/time manipulation: `formatDateTime()`, `addDays()`
- JSON handling: `json()`, `array()`, `createArray()`

```json
"Set_Variable": {
  "type": "SetVariable",
  "inputs": {
    "name": "formattedData",
    "value": "@{map(body('Parse_JSON'), item => {
      return {
        id: item.id,
        name: toUpper(item.name),
        date: formatDateTime(item.timestamp, 'yyyy-MM-dd')
      }
    })}"
  }
}
```

### 5. Parameters and Variables

- **Parameterize your workflows** for reusability across environments
- **Use variables for temporary values** within a workflow
- **Define clear parameter schemas** with default values and descriptions

```json
"parameters": {
  "apiEndpoint": {
    "type": "string",
    "defaultValue": "https://api.dev.example.com",
    "metadata": {
      "description": "The base URL for the API endpoint"
    }
  }
},
"variables": {
  "requestId": "@{guid()}",
  "processedItems": []
}
```

### 6. Control Flow

- **Use conditions** for branching logic
- **Implement parallel branches** for independent operations
- **Use foreach loops** with reasonable batch sizes for collections
- **Apply until loops** with proper exit conditions

```json
"Process_Items": {
  "type": "Foreach",
  "foreach": "@body('Get_Items')",
  "actions": {
    "Process_Single_Item": {
      "type": "Scope",
      "actions": { }
    }
  },
  "runAfter": {
    "Get_Items": ["Succeeded"]
  },
  "runtimeConfiguration": {
    "concurrency": {
      "repetitions": 10
    }
  }
}
```

### 7. Content and Message Handling

- **Validate message schemas** to ensure data integrity
- **Implement proper content type handling**
- **Use Parse JSON actions** to work with structured data

```json
"Parse_Response": {
  "type": "ParseJson",
  "inputs": {
    "content": "@body('HTTP_Request')",
    "schema": {
      "type": "object",
      "properties": {
        "id": {
          "type": "string"
        },
        "data": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": { }
          }
        }
      }
    }
  }
}
```

### 8. Security Best Practices

- **Use managed identities** when possible
- **Store secrets in Key Vault**
- **Implement least privilege access** for connections
- **Secure API endpoints** with authentication
- **Implement IP restrictions** for HTTP triggers
- **Apply data encryption** for sensitive data in parameters and messages
- **Use Azure RBAC** to control access to Logic Apps resources
- **Conduct regular security reviews** of workflows and connections

```json
"Get_Secret": {
  "type": "ApiConnection",
  "inputs": {
    "host": {
      "connection": {
        "name": "@parameters('$connections')['keyvault']['connectionId']"
      }
    },
    "method": "get",
    "path": "/secrets/@{encodeURIComponent('apiKey')}/value"
  }
},
"Call_Protected_API": {
  "type": "Http",
  "inputs": {
    "method": "POST",
    "uri": "https://api.example.com/protected",
    "headers": {
      "Content-Type": "application/json",
      "Authorization": "Bearer @{body('Get_Secret')?['value']}"
    },
    "body": {
      "data": "@variables('processedData')"
    }
  },
  "authentication": {
    "type": "ManagedServiceIdentity"
  },
  "runAfter": {
    "Get_Secret": ["Succeeded"]
  }
}
```

## Performance Optimization

- **Minimize unnecessary actions**
- **Use batch operations** when available
- **Optimize expressions** to reduce complexity
- **Configure appropriate timeout values**
- **Implement pagination** for large data sets
- **Implement concurrency control** for parallelizable operations

```json
"Process_Items": {
  "type": "Foreach",
  "foreach": "@body('Get_Items')",
  "actions": {
    "Process_Single_Item": {
      "type": "Scope",
      "actions": { }
    }
  },
  "runAfter": {
    "Get_Items": ["Succeeded"]
  },
  "runtimeConfiguration": {
    "concurrency": {
      "repetitions": 10
    }
  }
}
```

### Workflow Design Best Practices

- **Limit workflows to 50 actions or less** for optimal designer performance
- **Split complex business logic** into multiple smaller workflows when necessary
- **Use deployment slots** for mission-critical logic apps that require zero downtime deployments
- **Avoid hardcoded properties** in trigger and action definitions
- **Add descriptive comments** to provide context about trigger and action definitions
- **Use built-in operations** when available instead of shared connectors for better performance
- **Use an Integration Account** for B2B scenarios and EDI message processing
- **Reuse workflow templates** for standard patterns across your organization
- **Avoid deep nesting** of scopes and actions to maintain readability

### Monitoring and Observability

- **Configure diagnostic settings** to capture workflow runs and metrics
- **Add tracking IDs** to correlate related workflow runs
- **Implement comprehensive logging** with appropriate detail levels
- **Set up alerts** for workflow failures and performance degradation
- **Use Application Insights** for end-to-end tracing and monitoring

## Platform Types and Considerations

## Common Integration Patterns

### Architectural Patterns
- **Mediator Pattern**: Use Logic Apps as an orchestration layer between systems
- **Content-Based Routing**: Route messages based on content to different destinations
- **Message Transformation**: Transform messages between formats (JSON, XML, EDI, etc.)
- **Scatter-Gather**: Distribute work in parallel and aggregate results
- **Protocol Bridging**: Connect systems with different protocols (REST, SOAP, FTP, etc.)
- **Claim Check**: Store large payloads externally in blob storage or databases
- **Saga Pattern**: Manage distributed transactions with compensating actions for failures
- **Choreography Pattern**: Coordinate multiple services without a central orchestrator

### Action Patterns
- **Asynchronous Processing Pattern**: For long-running operations
  ```json
  "LongRunningAction": {
    "type": "Http",
    "inputs": {
      "method": "POST",
      "uri": "https://api.example.com/longrunning",
      "body": { "data": "@triggerBody()" }
    },
    "retryPolicy": {
      "type": "fixed",
      "count": 3,
      "interval": "PT30S"
    }
  }
  ```

- **Webhook Pattern**: For callback-based processing
  ```json
  "WebhookAction": {
    "type": "ApiConnectionWebhook",
    "inputs": {
      "host": {
        "connection": {
          "name": "@parameters('$connections')['servicebus']['connectionId']"
        }
      },
      "body": {
        "content": "@triggerBody()"
      },
      "path": "/subscribe/topics/@{encodeURIComponent('mytopic')}/subscriptions/@{encodeURIComponent('mysubscription')}"
    }
  }
  ```

### Enterprise Integration Patterns
- **B2B Message Exchange**: Exchange EDI documents between trading partners (AS2, X12, EDIFACT)
- **Integration Account**: Use for storing and managing B2B artifacts (agreements, schemas, maps)
- **Rules Engine**: Implement complex business rules using the Azure Logic Apps Rules Engine
- **Message Validation**: Validate messages against schemas for compliance and data integrity
- **Transaction Processing**: Process business transactions with compensating transactions for rollback

## DevOps and CI/CD for Logic Apps

### Source Control and Versioning

- **Store Logic App definitions in source control** (Git, Azure DevOps, GitHub)
- **Use ARM templates** for deployment to multiple environments
- **Implement branching strategies** appropriate for your release cadence
- **Version your Logic Apps** using tags or version properties

### Automated Deployment

- **Use Azure DevOps pipelines** or GitHub Actions for automated deployments
- **Implement parameterization** for environment-specific values
- **Use deployment slots** for zero-downtime deployments
- **Include post-deployment validation** tests in your CI/CD pipeline

```yaml
# Example Azure DevOps YAML pipeline for Logic App deployment
trigger:
  branches:
    include:
    - main
    - release/*

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: AzureResourceManagerTemplateDeployment@3
  inputs:
    deploymentScope: 'Resource Group'
    azureResourceManagerConnection: 'Your-Azure-Connection'
    subscriptionId: '$(subscriptionId)'
    action: 'Create Or Update Resource Group'
    resourceGroupName: '$(resourceGroupName)'
    location: '$(location)'
    templateLocation: 'Linked artifact'
    csmFile: '$(System.DefaultWorkingDirectory)/arm-templates/logicapp-template.json'
    csmParametersFile: '$(System.DefaultWorkingDirectory)/arm-templates/logicapp-parameters-$(Environment).json'
    deploymentMode: 'Incremental'
```

## Practical Logic App Examples

### HTTP Request Handler with API Integration

This example demonstrates a Logic App that accepts an HTTP request, validates the input data, calls an external API, transforms the response, and returns a formatted result.

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "actions": {
      "Validate_Input": {
        "type": "If",
        "expression": {
          "and": [
            {
              "not": {
                "equals": [
                  "@triggerBody()?['customerId']",
                  null
                ]
              }
            },
            {
              "not": {
                "equals": [
                  "@triggerBody()?['requestType']",
                  null
                ]
              }
            }
          ]
        },
        "actions": {
          "Get_Customer_Data": {
            "type": "Http",
            "inputs": {
              "method": "GET",
              "uri": "https://api.example.com/customers/@{triggerBody()?['customerId']}",
              "headers": {
                "Content-Type": "application/json",
                "Authorization": "Bearer @{body('Get_API_Key')?['value']}"
              }
            },
            "runAfter": {
              "Get_API_Key": [
                "Succeeded"
              ]
            }
          },
          "Get_API_Key": {
            "type": "ApiConnection",
            "inputs": {
              "host": {
                "connection": {
                  "name": "@parameters('$connections')['keyvault']['connectionId']"
                }
              },
              "method": "get",
              "path": "/secrets/@{encodeURIComponent('apiKey')}/value"
            }
          },
          "Parse_Customer_Response": {
            "type": "ParseJson",
            "inputs": {
              "content": "@body('Get_Customer_Data')",
              "schema": {
                "type": "object",
                "properties": {
                  "id": { "type": "string" },
                  "name": { "type": "string" },
                  "email": { "type": "string" },
                  "status": { "type": "string" },
                  "createdDate": { "type": "string" },
                  "orders": {
                    "type": "array",
                    "items": {
                      "type": "object",
                      "properties": {
                        "orderId": { "type": "string" },
                        "orderDate": { "type": "string" },
                        "amount": { "type": "number" }
                      }
                    }
                  }
                }
              }
            },
            "runAfter": {
              "Get_Customer_Data": [
                "Succeeded"
              ]
            }
          },
          "Switch_Request_Type": {
            "type": "Switch",
            "expression": "@triggerBody()?['requestType']",
            "cases": {
              "Profile": {
                "actions": {
                  "Prepare_Profile_Response": {
                    "type": "SetVariable",
                    "inputs": {
                      "name": "responsePayload",
                      "value": {
                        "customerId": "@body('Parse_Customer_Response')?['id']",
                        "customerName": "@body('Parse_Customer_Response')?['name']",
                        "email": "@body('Parse_Customer_Response')?['email']",
                        "status": "@body('Parse_Customer_Response')?['status']",
                        "memberSince": "@formatDateTime(body('Parse_Customer_Response')?['createdDate'], 'yyyy-MM-dd')"
                      }
                    }
                  }
                }
              },
              "OrderSummary": {
                "actions": {
                  "Calculate_Order_Statistics": {
                    "type": "Compose",
                    "inputs": {
                      "totalOrders": "@length(body('Parse_Customer_Response')?['orders'])",
                      "totalSpent": "@sum(body('Parse_Customer_Response')?['orders'], item => item.amount)",
                      "averageOrderValue": "@if(greater(length(body('Parse_Customer_Response')?['orders']), 0), div(sum(body('Parse_Customer_Response')?['orders'], item => item.amount), length(body('Parse_Customer_Response')?['orders'])), 0)",
                      "lastOrderDate": "@if(greater(length(body('Parse_Customer_Response')?['orders']), 0), max(body('Parse_Customer_Response')?['orders'], item => item.orderDate), '')"
                    }
                  },
                  "Prepare_Order_Response": {
                    "type": "SetVariable",
                    "inputs": {
                      "name": "responsePayload",
                      "value": {
                        "customerId": "@body('Parse_Customer_Response')?['id']",
                        "customerName": "@body('Parse_Customer_Response')?['name']",
                        "orderStats": "@outputs('Calculate_Order_Statistics')"
                      }
                    },
                    "runAfter": {
                      "Calculate_Order_Statistics": [
                        "Succeeded"
                      ]
                    }
                  }
                }
              }
            },
            "default": {
              "actions": {
                "Set_Default_Response": {
                  "type": "SetVariable",
                  "inputs": {
                    "name": "responsePayload",
                    "value": {
                      "error": "Invalid request type specified",
                      "validTypes": [
                        "Profile",
                        "OrderSummary"
                      ]
                    }
                  }
                }
              }
            },
            "runAfter": {
              "Parse_Customer_Response": [
                "Succeeded"
              ]
            }
          },
          "Log_Successful_Request": {
            "type": "ApiConnection",
            "inputs": {
              "host": {
                "connection": {
                  "name": "@parameters('$connections')['applicationinsights']['connectionId']"
                }
              },
              "method": "post",
              "body": {
                "LogType": "ApiRequestSuccess",
                "CustomerId": "@triggerBody()?['customerId']",
                "RequestType": "@triggerBody()?['requestType']",
                "ProcessingTime": "@workflow()['run']['duration']"
              }
            },
            "runAfter": {
              "Switch_Request_Type": [
                "Succeeded"
              ]
            }
          },
          "Return_Success_Response": {
            "type": "Response",
            "kind": "Http",
            "inputs": {
              "statusCode": 200,
              "body": "@variables('responsePayload')",
              "headers": {
                "Content-Type": "application/json"
              }
            },
            "runAfter": {
              "Log_Successful_Request": [
                "Succeeded"
              ]
            }
          }
        },
        "else": {
          "actions": {
            "Return_Validation_Error": {
              "type": "Response",
              "kind": "Http",
              "inputs": {
                "statusCode": 400,
                "body": {
                  "error": "Invalid request",
                  "message": "Request must include customerId and requestType",
                  "timestamp": "@utcNow()"
                }
              }
            }
          }
        },
        "runAfter": {
          "Initialize_Response_Variable": [
            "Succeeded"
          ]
        }
      },
      "Initialize_Response_Variable": {
        "type": "InitializeVariable",
        "inputs": {
          "variables": [
            {
              "name": "responsePayload",
              "type": "object",
              "value": {}
            }
          ]
        }
      }
    },
    "contentVersion": "1.0.0.0",
    "outputs": {},
    "parameters": {
      "$connections": {
        "defaultValue": {},
        "type": "Object"
      }
    },
    "triggers": {
      "manual": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "customerId": {
                "type": "string"
              },
              "requestType": {
                "type": "string",
                "enum": [
                  "Profile",
                  "OrderSummary"
                ]
              }
            }
          }
        }
      }
    }
  },
  "parameters": {
    "$connections": {
      "value": {
        "keyvault": {
          "connectionId": "/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.Web/connections/keyvault",
          "connectionName": "keyvault",
          "id": "/subscriptions/{subscription-id}/providers/Microsoft.Web/locations/{location}/managedApis/keyvault"
        },
        "applicationinsights": {
          "connectionId": "/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.Web/connections/applicationinsights",
          "connectionName": "applicationinsights",
          "id": "/subscriptions/{subscription-id}/providers/Microsoft.Web/locations/{location}/managedApis/applicationinsights"
        }
      }
    }
  }
}
```

### Event-Driven Process with Error Handling

This example demonstrates a Logic App that processes events from Azure Service Bus, handles the message processing with robust error handling, and implements the retry pattern for resilience.

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "actions": {
      "Parse_Message": {
        "type": "ParseJson",
        "inputs": {
          "content": "@triggerBody()?['ContentData']",
          "schema": {
            "type": "object",
            "properties": {
              "eventId": { "type": "string" },
              "eventType": { "type": "string" },
              "eventTime": { "type": "string" },
              "dataVersion": { "type": "string" },
              "data": {
                "type": "object",
                "properties": {
                  "orderId": { "type": "string" },
                  "customerId": { "type": "string" },
                  "items": {
                    "type": "array",
                    "items": {
                      "type": "object",
                      "properties": {
                        "productId": { "type": "string" },
                        "quantity": { "type": "integer" },
                        "unitPrice": { "type": "number" }
                      }
                    }
                  }
                }
              }
            }
          }
        },
        "runAfter": {}
      },
      "Try_Process_Order": {
        "type": "Scope",
        "actions": {
          "Get_Customer_Details": {
            "type": "Http",
            "inputs": {
              "method": "GET",
              "uri": "https://api.example.com/customers/@{body('Parse_Message')?['data']?['customerId']}",
              "headers": {
                "Content-Type": "application/json",
                "Authorization": "Bearer @{body('Get_API_Key')?['value']}"
              }
            },
            "runAfter": {
              "Get_API_Key": [
                "Succeeded"
              ]
            },
            "retryPolicy": {
              "type": "exponential",
              "count": 5,
              "interval": "PT10S",
              "minimumInterval": "PT5S",
              "maximumInterval": "PT1H"
            }
          },
          "Get_API_Key": {
            "type": "ApiConnection",
            "inputs": {
              "host": {
                "connection": {
                  "name": "@parameters('$connections')['keyvault']['connectionId']"
                }
              },
              "method": "get",
              "path": "/secrets/@{encodeURIComponent('apiKey')}/value"
            }
          },
          "Validate_Stock": {
            "type": "Foreach",
            "foreach": "@body('Parse_Message')?['data']?['items']",
            "actions": {
              "Check_Product_Stock": {
                "type": "Http",
                "inputs": {
                  "method": "GET",
                  "uri": "https://api.example.com/inventory/@{items('Validate_Stock')?['productId']}",
                  "headers": {
                    "Content-Type": "application/json",
                    "Authorization": "Bearer @{body('Get_API_Key')?['value']}"
                  }
                },
                "retryPolicy": {
                  "type": "fixed",
                  "count": 3,
                  "interval": "PT15S"
                }
              },
              "Verify_Availability": {
                "type": "If",
                "expression": {
                  "and": [
                    {
                      "greater": [
                        "@body('Check_Product_Stock')?['availableStock']",
                        "@items('Validate_Stock')?['quantity']"
                      ]
                    }
                  ]
                },
                "actions": {
                  "Add_To_Valid_Items": {
                    "type": "AppendToArrayVariable",
                    "inputs": {
                      "name": "validItems",
                      "value": {
                        "productId": "@items('Validate_Stock')?['productId']",
                        "quantity": "@items('Validate_Stock')?['quantity']",
                        "unitPrice": "@items('Validate_Stock')?['unitPrice']",
                        "availableStock": "@body('Check_Product_Stock')?['availableStock']"
                      }
                    }
                  }
                },
                "else": {
                  "actions": {
                    "Add_To_Invalid_Items": {
                      "type": "AppendToArrayVariable",
                      "inputs": {
                        "name": "invalidItems",
                        "value": {
                          "productId": "@items('Validate_Stock')?['productId']",
                          "requestedQuantity": "@items('Validate_Stock')?['quantity']",
                          "availableStock": "@body('Check_Product_Stock')?['availableStock']",
                          "reason": "Insufficient stock"
                        }
                      }
                    }
                  }
                },
                "runAfter": {
                  "Check_Product_Stock": [
                    "Succeeded"
                  ]
                }
              }
            },
            "runAfter": {
              "Get_Customer_Details": [
                "Succeeded"
              ]
            }
          },
          "Check_Order_Validity": {
            "type": "If",
            "expression": {
              "and": [
                {
                  "equals": [
                    "@length(variables('invalidItems'))",
                    0
                  ]
                },
                {
                  "greater": [
                    "@length(variables('validItems'))",
                    0
                  ]
                }
              ]
            },
            "actions": {
              "Process_Valid_Order": {
                "type": "Http",
                "inputs": {
                  "method": "POST",
                  "uri": "https://api.example.com/orders",
                  "headers": {
                    "Content-Type": "application/json",
                    "Authorization": "Bearer @{body('Get_API_Key')?['value']}"
                  },
                  "body": {
                    "orderId": "@body('Parse_Message')?['data']?['orderId']",
                    "customerId": "@body('Parse_Message')?['data']?['customerId']",
                    "customerName": "@body('Get_Customer_Details')?['name']",
                    "items": "@variables('validItems')",
                    "processedTime": "@utcNow()",
                    "eventId": "@body('Parse_Message')?['eventId']"
                  }
                }
              },
              "Send_Order_Confirmation": {
                "type": "ApiConnection",
                "inputs": {
                  "host": {
                    "connection": {
                      "name": "@parameters('$connections')['office365']['connectionId']"
                    }
                  },
                  "method": "post",
                  "path": "/v2/Mail",
                  "body": {
                    "To": "@body('Get_Customer_Details')?['email']",
                    "Subject": "Order Confirmation: @{body('Parse_Message')?['data']?['orderId']}",
                    "Body": "<p>Dear @{body('Get_Customer_Details')?['name']},</p><p>Your order has been successfully processed.</p><p>Order ID: @{body('Parse_Message')?['data']?['orderId']}</p><p>Thank you for your business!</p>",
                    "Importance": "Normal",
                    "IsHtml": true
                  }
                },
                "runAfter": {
                  "Process_Valid_Order": [
                    "Succeeded"
                  ]
                }
              },
              "Complete_Message": {
                "type": "ApiConnection",
                "inputs": {
                  "host": {
                    "connection": {
                      "name": "@parameters('$connections')['servicebus']['connectionId']"
                    }
                  },
                  "method": "post",
                  "path": "/messages/complete",
                  "body": {
                    "lockToken": "@triggerBody()?['LockToken']",
                    "sessionId": "@triggerBody()?['SessionId']",
                    "queueName": "@parameters('serviceBusQueueName')"
                  }
                },
                "runAfter": {
                  "Send_Order_Confirmation": [
                    "Succeeded"
                  ]
                }
              }
            },
            "else": {
              "actions": {
                "Send_Invalid_Stock_Notification": {
                  "type": "ApiConnection",
                  "inputs": {
                    "host": {
                      "connection": {
                        "name": "@parameters('$connections')['office365']['connectionId']"
                      }
                    },
                    "method": "post",
                    "path": "/v2/Mail",
                    "body": {
                      "To": "@body('Get_Customer_Details')?['email']",
                      "Subject": "Order Cannot Be Processed: @{body('Parse_Message')?['data']?['orderId']}",
                      "Body": "<p>Dear @{body('Get_Customer_Details')?['name']},</p><p>We regret to inform you that your order cannot be processed due to insufficient stock for the following items:</p><p>@{join(variables('invalidItems'), '</p><p>')}</p><p>Please adjust your order and try again.</p>",
                      "Importance": "High",
                      "IsHtml": true
                    }
                  }
                },
                "Dead_Letter_Message": {
                  "type": "ApiConnection",
                  "inputs": {
                    "host": {
                      "connection": {
                        "name": "@parameters('$connections')['servicebus']['connectionId']"
                      }
                    },
                    "method": "post",
                    "path": "/messages/deadletter",
                    "body": {
                      "lockToken": "@triggerBody()?['LockToken']",
                      "sessionId": "@triggerBody()?['SessionId']",
                      "queueName": "@parameters('serviceBusQueueName')",
                      "deadLetterReason": "InsufficientStock",
                      "deadLetterDescription": "Order contained items with insufficient stock"
                    }
                  },
                  "runAfter": {
                    "Send_Invalid_Stock_Notification": [
                      "Succeeded"
                    ]
                  }
                }
              }
            },
            "runAfter": {
              "Validate_Stock": [
                "Succeeded"
              ]
            }
          }
        },
        "runAfter": {
          "Initialize_Variables": [
            "Succeeded"
          ]
        }
      },
      "Initialize_Variables": {
        "type": "InitializeVariable",
        "inputs": {
          "variables": [
            {
              "name": "validItems",
              "type": "array",
              "value": []
            },
            {
              "name": "invalidItems",
              "type": "array",
              "value": []
            }
          ]
        },
        "runAfter": {
          "Parse_Message": [
            "Succeeded"
          ]
        }
      },
      "Handle_Process_Error": {
        "type": "Scope",
        "actions": {
          "Log_Error_Details": {
            "type": "ApiConnection",
            "inputs": {
              "host": {
                "connection": {
                  "name": "@parameters('$connections')['applicationinsights']['connectionId']"
                }
              },
              "method": "post",
              "body": {
                "LogType": "OrderProcessingError",
                "EventId": "@body('Parse_Message')?['eventId']",
                "OrderId": "@body('Parse_Message')?['data']?['orderId']",
                "CustomerId": "@body('Parse_Message')?['data']?['customerId']",
                "ErrorDetails": "@result('Try_Process_Order')",
                "Timestamp": "@utcNow()"
              }
            }
          },
          "Abandon_Message": {
            "type": "ApiConnection",
            "inputs": {
              "host": {
                "connection": {
                  "name": "@parameters('$connections')['servicebus']['connectionId']"
                }
              },
              "method": "post",
              "path": "/messages/abandon",
              "body": {
                "lockToken": "@triggerBody()?['LockToken']",
                "sessionId": "@triggerBody()?['SessionId']",
                "queueName": "@parameters('serviceBusQueueName')"
              }
            },
            "runAfter": {
              "Log_Error_Details": [
                "Succeeded"
              ]
            }
          },
          "Send_Alert_To_Operations": {
            "type": "ApiConnection",
            "inputs": {
              "host": {
                "connection": {
                  "name": "@parameters('$connections')['office365']['connectionId']"
                }
              },
              "method": "post",
              "path": "/v2/Mail",
              "body": {
                "To": "operations@example.com",
                "Subject": "Order Processing Error: @{body('Parse_Message')?['data']?['orderId']}",
                "Body": "<p>An error occurred while processing an order:</p><p>Order ID: @{body('Parse_Message')?['data']?['orderId']}</p><p>Customer ID: @{body('Parse_Message')?['data']?['customerId']}</p><p>Error: @{result('Try_Process_Order')}</p>",
                "Importance": "High",
                "IsHtml": true
              }
            },
            "runAfter": {
              "Abandon_Message": [
                "Succeeded"
              ]
            }
          }
        },
        "runAfter": {
          "Try_Process_Order": [
            "Failed",
            "TimedOut"
          ]
        }
      }
    },
    "contentVersion": "1.0.0.0",
    "outputs": {},
    "parameters": {
      "$connections": {
        "defaultValue": {},
        "type": "Object"
      },
      "serviceBusQueueName": {
        "type": "string",
        "defaultValue": "orders"
      }
    },
    "triggers": {
      "When_a_message_is_received_in_a_queue": {
        "type": "ApiConnectionWebhook",
        "inputs": {
          "host": {
            "connection": {
              "name": "@parameters('$connections')['servicebus']['connectionId']"
            }
          },
          "body": {
            "isSessionsEnabled": true
          },
          "path": "/subscriptionListener",
          "queries": {
            "queueName": "@parameters('serviceBusQueueName')",
            "subscriptionType": "Main"
          }
        }
      }
    }
  },
  "parameters": {
    "$connections": {
      "value": {
        "keyvault": {
          "connectionId": "/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.Web/connections/keyvault",
          "connectionName": "keyvault",
          "id": "/subscriptions/{subscription-id}/providers/Microsoft.Web/locations/{location}/managedApis/keyvault"
        },
        "servicebus": {
          "connectionId": "/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.Web/connections/servicebus",
          "connectionName": "servicebus",
          "id": "/subscriptions/{subscription-id}/providers/Microsoft.Web/locations/{location}/managedApis/servicebus"
        },
        "office365": {
          "connectionId": "/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.Web/connections/office365",
          "connectionName": "office365",
          "id": "/subscriptions/{subscription-id}/providers/Microsoft.Web/locations/{location}/managedApis/office365"
        },
        "applicationinsights": {
          "connectionId": "/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.Web/connections/applicationinsights",
          "connectionName": "applicationinsights",
          "id": "/subscriptions/{subscription-id}/providers/Microsoft.Web/locations/{location}/managedApis/applicationinsights"
        }
      }
    }
  }
}
```

## Advanced Exception Handling and Monitoring

### Comprehensive Exception Handling Strategy

Implement a multi-layered exception handling approach for robust workflows:

1. **Preventative Measures**:
   - Use schema validation for all incoming messages
   - Implement defensive expression evaluations using `coalesce()` and `?` operators
   - Add pre-condition checks before critical operations

2. **Runtime Error Handling**:
   - Use structured error handling scopes with nested try/catch patterns
   - Implement circuit breaker patterns for external dependencies
   - Capture and handle specific error types differently

```json
"Process_With_Comprehensive_Error_Handling": {
  "type": "Scope",
  "actions": {
    "Try_Primary_Action": {
      "type": "Scope",
      "actions": {
        "Main_Operation": {
          "type": "Http",
          "inputs": { "method": "GET", "uri": "https://api.example.com/resource" }
        }
      }
    },
    "Handle_Connection_Errors": {
      "type": "Scope",
      "actions": {
        "Log_Connection_Error": {
          "type": "ApiConnection",
          "inputs": {
            "host": {
              "connection": {
                "name": "@parameters('$connections')['loganalytics']['connectionId']"
              }
            },
            "method": "post",
            "body": {
              "LogType": "ConnectionError",
              "ErrorCategory": "Network",
              "StatusCode": "@{result('Try_Primary_Action')?['outputs']?['Main_Operation']?['statusCode']}",
              "ErrorMessage": "@{result('Try_Primary_Action')?['error']?['message']}"
            }
          }
        },
        "Invoke_Fallback_Endpoint": {
          "type": "Http",
          "inputs": { "method": "GET", "uri": "https://fallback-api.example.com/resource" }
        }
      },
      "runAfter": {
        "Try_Primary_Action": ["Failed"]
      }
    },
    "Handle_Business_Logic_Errors": {
      "type": "Scope",
      "actions": {
        "Parse_Error_Response": {
          "type": "ParseJson",
          "inputs": {
            "content": "@outputs('Try_Primary_Action')?['Main_Operation']?['body']",
            "schema": {
              "type": "object",
              "properties": {
                "errorCode": { "type": "string" },
                "errorMessage": { "type": "string" }
              }
            }
          }
        },
        "Switch_On_Error_Type": {
          "type": "Switch",
          "expression": "@body('Parse_Error_Response')?['errorCode']",
          "cases": {
            "ResourceNotFound": {
              "actions": { "Create_Resource": { "type": "Http", "inputs": {} } }
            },
            "ValidationError": {
              "actions": { "Resubmit_With_Defaults": { "type": "Http", "inputs": {} } }
            },
            "PermissionDenied": {
              "actions": { "Elevate_Permissions": { "type": "Http", "inputs": {} } }
            }
          },
          "default": {
            "actions": { "Send_To_Support_Queue": { "type": "ApiConnection", "inputs": {} } }
          }
        }
      },
      "runAfter": {
        "Try_Primary_Action": ["Succeeded"]
      }
    }
  }
}
```

3. **Centralized Error Logging**:
   - Create a dedicated Logic App for error handling that other workflows can call
   - Log errors with correlation IDs for traceability across systems
   - Categorize errors by type and severity for better analysis

### Advanced Monitoring Architecture

Implement a comprehensive monitoring strategy that covers:

1. **Operational Monitoring**:
   - **Health Probes**: Create dedicated health check workflows
   - **Heartbeat Patterns**: Implement periodic check-ins to verify system health
   - **Dead Letter Handling**: Process and analyze failed messages

2. **Business Process Monitoring**:
   - **Business Metrics**: Track key business KPIs (order processing times, approval rates)
   - **SLA Monitoring**: Measure performance against service level agreements
   - **Correlated Tracing**: Implement end-to-end transaction tracking

3. **Alerting Strategy**:
   - **Multi-channel Alerts**: Configure alerts to appropriate channels (email, SMS, Teams)
   - **Severity-based Routing**: Route alerts based on business impact
   - **Alert Correlation**: Group related alerts to prevent alert fatigue

```json
"Monitor_Transaction_SLA": {
  "type": "Scope",
  "actions": {
    "Calculate_Processing_Time": {
      "type": "Compose",
      "inputs": "@{div(sub(ticks(utcNow()), ticks(triggerBody()?['startTime'])), 10000000)}"
    },
    "Check_SLA_Breach": {
      "type": "If",
      "expression": "@greater(outputs('Calculate_Processing_Time'), parameters('slaThresholdSeconds'))",
      "actions": {
        "Log_SLA_Breach": {
          "type": "ApiConnection",
          "inputs": {
            "host": {
              "connection": {
                "name": "@parameters('$connections')['loganalytics']['connectionId']"
              }
            },
            "method": "post",
            "body": {
              "LogType": "SLABreach",
              "TransactionId": "@{triggerBody()?['transactionId']}",
              "ProcessingTimeSeconds": "@{outputs('Calculate_Processing_Time')}",
              "SLAThresholdSeconds": "@{parameters('slaThresholdSeconds')}",
              "BreachSeverity": "@if(greater(outputs('Calculate_Processing_Time'), mul(parameters('slaThresholdSeconds'), 2)), 'Critical', 'Warning')"
            }
          }
        },
        "Send_SLA_Alert": {
          "type": "ApiConnection",
          "inputs": {
            "host": {
              "connection": {
                "name": "@parameters('$connections')['teams']['connectionId']"
              }
            },
            "method": "post",
            "body": {
              "notificationTitle": "SLA Breach Alert",
              "message": "Transaction @{triggerBody()?['transactionId']} exceeded SLA by @{sub(outputs('Calculate_Processing_Time'), parameters('slaThresholdSeconds'))} seconds",
              "channelId": "@{if(greater(outputs('Calculate_Processing_Time'), mul(parameters('slaThresholdSeconds'), 2)), parameters('criticalAlertChannelId'), parameters('warningAlertChannelId'))}"
            }
          }
        }
      }
    }
  }
}
```

## API Management Integration

Integrate Logic Apps with Azure API Management for enhanced security, governance, and management:

### API Management Frontend

- **Expose Logic Apps via API Management**:
  - Create API definitions for Logic App HTTP triggers
  - Apply consistent URL structures and versioning
  - Implement API policies for security and transformation

### Policy Templates for Logic Apps

```xml
<!-- Logic App API Policy Example -->
<policies>
  <inbound>
    <!-- Authentication -->
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized">
      <openid-config url="https://login.microsoftonline.com/{tenant-id}/.well-known/openid-configuration" />
      <required-claims>
        <claim name="aud" match="any">
          <value>api://mylogicapp</value>
        </claim>
      </required-claims>
    </validate-jwt>
    
    <!-- Rate limiting -->
    <rate-limit calls="5" renewal-period="60" />
    
    <!-- Request transformation -->
    <set-header name="Correlation-Id" exists-action="override">
      <value>@(context.RequestId)</value>
    </set-header>
    
    <!-- Logging -->
    <log-to-eventhub logger-id="api-logger">
      @{
        return new JObject(
          new JProperty("correlationId", context.RequestId),
          new JProperty("api", context.Api.Name),
          new JProperty("operation", context.Operation.Name),
          new JProperty("user", context.User.Email),
          new JProperty("ip", context.Request.IpAddress)
        ).ToString();
      }
    </log-to-eventhub>
  </inbound>
  <backend>
    <forward-request />
  </backend>
  <outbound>
    <!-- Response transformation -->
    <set-header name="X-Powered-By" exists-action="delete" />
  </outbound>
  <on-error>
    <base />
  </on-error>
</policies>
```

### Workflow as API Pattern

- **Implement Workflow as API pattern**:
  - Design Logic Apps specifically as API backends
  - Use request triggers with OpenAPI schemas
  - Apply consistent response patterns
  - Implement proper status codes and error handling

```json
"triggers": {
  "manual": {
    "type": "Request",
    "kind": "Http",
    "inputs": {
      "schema": {
        "$schema": "http://json-schema.org/draft-04/schema#",
        "type": "object",
        "properties": {
          "customerId": {
            "type": "string",
            "description": "The unique identifier for the customer"
          },
          "requestType": {
            "type": "string",
            "enum": ["Profile", "OrderSummary"],
            "description": "The type of request to process"
          }
        },
        "required": ["customerId", "requestType"]
      },
      "method": "POST"
    }
  }
}
```

## Versioning Strategies

Implement robust versioning approaches for Logic Apps:

### Versioning Patterns

1. **URI Path Versioning**:
   - Include version in HTTP trigger path (/api/v1/resource)
   - Maintain separate Logic Apps for each major version

2. **Parameter Versioning**:
   - Add version parameter to workflow definitions
   - Use conditional logic based on version parameter

3. **Side-by-Side Versioning**:
   - Deploy new versions alongside existing ones
   - Implement traffic routing between versions

### Version Migration Strategy

```json
"actions": {
  "Check_Request_Version": {
    "type": "Switch",
    "expression": "@triggerBody()?['apiVersion']",
    "cases": {
      "1.0": {
        "actions": {
          "Process_V1_Format": {
            "type": "Scope",
            "actions": { }
          }
        }
      },
      "2.0": {
        "actions": {
          "Process_V2_Format": {
            "type": "Scope",
            "actions": { }
          }
        }
      }
    },
    "default": {
      "actions": {
        "Return_Version_Error": {
          "type": "Response",
          "kind": "Http",
          "inputs": {
            "statusCode": 400,
            "body": {
              "error": "Unsupported API version",
              "supportedVersions": ["1.0", "2.0"]
            }
          }
        }
      }
    }
  }
}
```

### ARM Template Deployment for Different Versions

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "logicAppName": {
      "type": "string",
      "metadata": {
        "description": "Base name of the Logic App"
      }
    },
    "version": {
      "type": "string",
      "metadata": {
        "description": "Version of the Logic App to deploy"
      },
      "allowedValues": ["v1", "v2", "v3"]
    }
  },
  "variables": {
    "fullLogicAppName": "[concat(parameters('logicAppName'), '-', parameters('version'))]",
    "workflowDefinitionMap": {
      "v1": "[variables('v1Definition')]",
      "v2": "[variables('v2Definition')]",
      "v3": "[variables('v3Definition')]"
    },
    "v1Definition": {},
    "v2Definition": {},
    "v3Definition": {}
  },
  "resources": [
    {
      "type": "Microsoft.Logic/workflows",
      "apiVersion": "2019-05-01",
      "name": "[variables('fullLogicAppName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "definition": "[variables('workflowDefinitionMap')[parameters('version')]]"
      }
    }
  ]
}
```

## Cost Optimization Techniques

Implement strategies to optimize the cost of Logic Apps solutions:

### Logic Apps Standard (Workflow) Cost Optimization

1. **App Service Plan Selection**:
   - Right-size App Service Plans for workload requirements
   - Implement auto-scaling based on load patterns
   - Consider reserved instances for predictable workloads

2. **Resource Sharing**:
   - Consolidate workflows in shared App Service Plans
   - Implement shared connections and integration resources
   - Use integration accounts efficiently

### Cost Monitoring and Governance

```json
"Monitor_Execution_Costs": {
  "type": "ApiConnection",
  "inputs": {
    "host": {
      "connection": {
        "name": "@parameters('$connections')['loganalytics']['connectionId']"
      }
    },
    "method": "post",
    "body": {
      "LogType": "WorkflowCostMetrics",
      "WorkflowName": "@{workflow().name}",
      "ExecutionId": "@{workflow().run.id}",
      "ActionCount": "@{length(workflow().run.actions)}",
      "TriggerType": "@{workflow().triggers[0].kind}",
      "DataProcessedBytes": "@{workflow().run.transferred}",
      "ExecutionDurationSeconds": "@{div(workflow().run.duration, 'PT1S')}",
      "Timestamp": "@{utcNow()}"
    }
  },
  "runAfter": {
    "Main_Workflow_Actions": ["Succeeded", "Failed", "TimedOut"]
  }
}
```

## Enhanced Security Practices

Implement comprehensive security measures for Logic Apps workflows:

### Sensitive Data Handling

1. **Data Classification and Protection**:
   - Identify and classify sensitive data in workflows
   - Implement masking for sensitive data in logs and monitoring
   - Apply encryption for data at rest and in transit

2. **Secure Parameter Handling**:
   - Use Azure Key Vault for all secrets and credentials
   - Implement dynamic parameter resolution at runtime
   - Apply parameter encryption for sensitive values

```json
"actions": {
  "Get_Database_Credentials": {
    "type": "ApiConnection",
    "inputs": {
      "host": {
        "connection": {
          "name": "@parameters('$connections')['keyvault']['connectionId']"
        }
      },
      "method": "get",
      "path": "/secrets/@{encodeURIComponent('database-connection-string')}/value"
    }
  },
  "Execute_Database_Query": {
    "type": "ApiConnection",
    "inputs": {
      "host": {
        "connection": {
          "name": "@parameters('$connections')['sql']['connectionId']"
        }
      },
      "method": "post",
      "path": "/datasets/default/query",
      "body": {
        "query": "SELECT * FROM Customers WHERE CustomerId = @CustomerId",
        "parameters": {
          "CustomerId": "@triggerBody()?['customerId']"
        },
        "connectionString": "@body('Get_Database_Credentials')?['value']"
      }
    },
    "runAfter": {
      "Get_Database_Credentials": ["Succeeded"]
    }
  }
}
```

### Advanced Identity and Access Controls

1. **Fine-grained Access Control**:
   - Implement custom roles for Logic Apps management
   - Apply principle of least privilege for connections
   - Use managed identities for all Azure service access

2. **Access Reviews and Governance**:
   - Implement regular access reviews for Logic Apps resources
   - Apply Just-In-Time access for administrative operations
   - Audit all access and configuration changes

3. **Network Security**:
   - Implement network isolation using private endpoints
   - Apply IP restrictions for trigger endpoints
   - Use Virtual Network integration for Logic Apps Standard

```json
{
  "resources": [
    {
      "type": "Microsoft.Logic/workflows",
      "apiVersion": "2019-05-01",
      "name": "[parameters('logicAppName')]",
      "location": "[parameters('location')]",
      "identity": {
        "type": "SystemAssigned"
      },
      "properties": {
        "accessControl": {
          "triggers": {
            "allowedCallerIpAddresses": [
              {
                "addressRange": "13.91.0.0/16"
              },
              {
                "addressRange": "40.112.0.0/13"
              }
            ]
          },
          "contents": {
            "allowedCallerIpAddresses": [
              {
                "addressRange": "13.91.0.0/16"
              },
              {
                "addressRange": "40.112.0.0/13"
              }
            ]
          },
          "actions": {
            "allowedCallerIpAddresses": [
              {
                "addressRange": "13.91.0.0/16"
              },
              {
                "addressRange": "40.112.0.0/13"
              }
            ]
          }
        },
        "definition": {}
      }
    }
  ]
}
```

## Additional Resources

- [Azure Logic Apps Documentation](https://docs.microsoft.com/en-us/azure/logic-apps/)
- [Workflow Definition Language Schema](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language)
- [Enterprise Integration Patterns](https://docs.microsoft.com/en-us/azure/logic-apps/enterprise-integration-overview)
- [Logic Apps B2B Documentation](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-enterprise-integration-b2b)
- [Azure Logic Apps Limits and Configuration](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-limits-and-config)
- [Logic Apps Performance Optimization](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-performance-optimization)
- [Logic Apps Security Overview](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-securing-a-logic-app)
- [API Management and Logic Apps Integration](https://docs.microsoft.com/en-us/azure/api-management/api-management-create-api-logic-app)
- [Logic Apps Standard Networking](https://docs.microsoft.com/en-us/azure/logic-apps/connect-virtual-network-vnet-isolated-environment)