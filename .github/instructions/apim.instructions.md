---
description: 'Best practices for developing, securing, monitoring, and deploying APIs to Azure API Management'
applyTo: "**/src/apim/**/*,**/infra/**/*"
---

# Azure API Management Development and Deployment Best Practices

## Overview

This guide provides comprehensive best practices for developing, securing, monitoring, and deploying APIs to **existing Azure API Management instances** using GitHub Actions and Infrastructure as Code (Bicep). Focus is on policy-driven security, automated deployment, and enterprise-grade API governance.

## Prerequisites

- Existing Azure API Management instance
- GitHub repository with Actions enabled
- Azure service principal or workload identity for deployment
- Bicep CLI and Azure CLI installed for local development

## Project Structure

```
src/
├── apim/
│   ├── policies/
│   │   ├── global/
│   │   │   └── policy.xml
│   │   ├── products/
│   │   │   └── [product-name]/
│   │   │       └── policy.xml
│   │   ├── apis/
│   │   │   └── [api-name]/
│   │   │       ├── policy.xml
│   │   │       └── operations/
│   │   │           └── [operation-name]/
│   │   │               └── policy.xml
│   │   └── fragments/
│   │       └── [fragment-name].xml
│   ├── schemas/
│   │   └── [schema-name].json
│   ├── templates/
│   │   └── [template-name].xml
│   └── backends/
│       └── [backend-name].json
infra/
├── modules/
│   ├── apim/
│   │   ├── apis.bicep
│   │   ├── policies.bicep
│   │   ├── products.bicep
│   │   └── backends.bicep
│   └── monitoring/
│       └── logging.bicep
└── main.bicep
```

## Development Guidelines
- Each API developed needs to have a OpenAPI definitions in `/src/apim/schemas/`
- Policies handle authentication, routing, and parameter extraction
- Backend service configuration points to Logic Apps Standard with SAS tokens
- Use policy fragments for reusable components like authentication and error handling
- Follow naming conventions for resources to ensure uniqueness and clarity
- Ensure all policies are validated using XML schema validation tools
- Use single quotes within double-quoted conditions in XML policies

## Security Best Practices

### Authentication and Authorization

#### 1. OAuth 2.0 and Azure AD Integration
```xml
<!-- Use validate-azure-ad-token policy for Microsoft Entra ID authentication -->
<policies>
    <inbound>
        <validate-azure-ad-token tenant-id="{{tenant-id}}" 
                                 client-application-ids="{{client-app-ids}}"
                                 failed-validation-httpcode="401"
                                 failed-validation-error-message="Unauthorized">
            <audiences>
                <audience>api://your-api-id</audience>
            </audiences>
            <issuers>
                <issuer>https://sts.windows.net/{{tenant-id}}/</issuer>
            </issuers>
            <required-claims>
                <claim name="scp" match="all">
                    <value>api.read</value>
                </claim>
            </required-claims>
        </validate-azure-ad-token>
    </inbound>
</policies>
```

#### 2. JWT Token Validation
```xml
<!-- Configure JWT validation with proper security settings -->
<validate-jwt header-name="Authorization" 
              failed-validation-httpcode="401"
              failed-validation-error-message="Invalid JWT token"
              require-expiration-time="true"
              require-signed-tokens="true">
    <openid-config url="https://login.microsoftonline.com/{{tenant-id}}/v2.0/.well-known/openid_configuration" />
    <audiences>
        <audience>your-api-audience</audience>
    </audiences>
    <issuers>
        <issuer>https://sts.windows.net/{{tenant-id}}/</issuer>
    </issuers>
    <required-claims>
        <claim name="scope" match="any">
            <value>read</value>
            <value>write</value>
        </claim>
    </required-claims>
</validate-jwt>
```

#### 3. Client Certificate Authentication
```xml
<!-- Validate client certificates for enhanced security -->
<authentication-certificate thumbprint="{{certificate-thumbprint}}"
                            certificate-id="{{certificate-id}}"
                            body="@(context.Request.Certificate?.Thumbprint)"
                            password="{{certificate-password}}" />
```

### IP Filtering and Network Security
```xml
<!-- Restrict API access to specific IP addresses or ranges -->
<ip-filter action="allow">
    <address-range from="10.0.0.0" to="10.255.255.255" />
    <address>192.168.1.100</address>
</ip-filter>
```

### Input Validation and Content Security
```xml
<!-- Validate request content against OpenAPI schema -->
<validate-content unspecified-content-type-action="prevent"
                  max-size="102400"
                  size-exceeded-action="prevent">
    <content type="application/json" 
             validate-as="json" 
             action="prevent"
             allow-additional-properties="false" />
</validate-content>

<!-- Validate parameters against OpenAPI specification -->
<validate-parameters specified-parameter-action="prevent"
                     unspecified-parameter-action="prevent" />

<!-- Validate headers -->
<validate-headers specified-header-action="ignore"
                  unspecified-header-action="prevent" />
```

## Rate Limiting and Throttling Policies

### Global Rate Limiting
```xml
<!-- Apply rate limits at the global level -->
<rate-limit calls="1000" renewal-period="60" />
```

### Advanced Rate Limiting by Key
```xml
<!-- Rate limit by user/subscription with header exposure -->
<rate-limit-by-key calls="100"
                   renewal-period="60"
                   counter-key="@(context.Request.IpAddress)"
                   increment-condition="@(context.Response.StatusCode >= 200 && context.Response.StatusCode < 400)"
                   remaining-calls-header-name="X-RateLimit-Remaining"
                   remaining-calls-variable-name="remainingCallsPerIP"
                   total-calls-header-name="X-RateLimit-Limit" />
```

### Quota Management
```xml
<!-- Long-term quota enforcement -->
<quota-by-key calls="10000"
              bandwidth="40000"
              renewal-period="86400"
              counter-key="@(context.Request.Headers.GetValueOrDefault('Authorization','').AsJwt()?.Subject)" />
```

### Advanced Throttling for Sensitive Operations
```xml
<!-- Stricter limits for authentication endpoints -->
<choose>
    <when condition="@(context.Request.Url.Path.Contains('/auth/'))">
        <rate-limit-by-key calls="5"
                           renewal-period="300"
                           counter-key="@(context.Request.IpAddress)"
                           increment-condition="@(context.Response.StatusCode == 401 || context.Response.StatusCode == 403)" />
    </when>
</choose>
```

## Caching Strategies

### Response Caching
```xml
<!-- Cache successful responses for performance optimization -->
<cache-lookup vary-by-developer="false"
              vary-by-developer-groups="false"
              downstream-caching-type="none"
              must-revalidate="false"
              caching-type="internal" />

<cache-store duration="3600"
             cache-response="true"
             caching-type="internal" />
```

### Semantic Caching for AI APIs
```xml
<!-- Advanced semantic caching for Azure OpenAI APIs -->
<azure-openai-semantic-cache-lookup score-threshold="0.95" />
<azure-openai-semantic-cache-store duration="3600" />
```

## Monitoring and Observability

### Application Insights Integration
```xml
<!-- Custom telemetry and metrics -->
<emit-metric name="api-calls-total"
             value="1"
             namespace="MyAPI">
    <dimension name="operation" value="@(context.Operation.Name)" />
    <dimension name="status" value="@(context.Response.StatusCode.ToString())" />
</emit-metric>

<!-- Structured logging for debugging -->
<trace source="MyAPI"
       severity="information"
       message="@($"Request processed: {context.Operation.Name} - Status: {context.Response.StatusCode}")">
    <metadata name="correlationId" value="@(context.RequestId)" />
    <metadata name="userId" value="@(context.Request.Headers.GetValueOrDefault('Authorization','').AsJwt()?.Subject)" />
</trace>
```

### Event Hub Integration for Advanced Analytics
```xml
<!-- Send custom events to Event Hub for real-time analytics -->
<log-to-eventhub logger-id="event-hub-logger"
                 partition-id="0">
    <message>
        @{
            var jwt = context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt();
            return new JObject(
                new JProperty("timestamp", DateTime.UtcNow),
                new JProperty("operation", context.Operation.Name),
                new JProperty("userId", jwt?.Subject),
                new JProperty("responseTime", context.Elapsed.TotalMilliseconds),
                new JProperty("statusCode", context.Response.StatusCode)
            ).ToString();
        }
    </message>
</log-to-eventhub>
```

## Error Handling and Resilience

### Global Error Handling
```xml
<!-- Comprehensive error handling with security-conscious responses -->
<on-error>
    <set-variable name="errorDetails" 
                  value="@{
                      return new JObject(
                          new JProperty("timestamp", DateTime.UtcNow),
                          new JProperty("requestId", context.RequestId),
                          new JProperty("error", context.LastError?.Message ?? "Unknown error"),
                          new JProperty("operation", context.Operation?.Name)
                      );
                  }" />
    
    <choose>
        <when condition="@(context.LastError?.Source == "validate-jwt")">
            <return-response>
                <set-status code="401" reason="Unauthorized" />
                <set-header name="Content-Type" value="application/json" />
                <set-body>@{
                    return new JObject(
                        new JProperty("error", "invalid_token"),
                        new JProperty("error_description", "The access token is invalid or expired"),
                        new JProperty("requestId", context.RequestId)
                    ).ToString();
                }</set-body>
            </return-response>
        </when>
        <when condition="@(context.LastError?.Source == "rate-limit" || context.LastError?.Source == "quota")">
            <return-response>
                <set-status code="429" reason="Too Many Requests" />
                <set-header name="Content-Type" value="application/json" />
                <set-header name="Retry-After" value="60" />
                <set-body>@{
                    return new JObject(
                        new JProperty("error", "rate_limit_exceeded"),
                        new JProperty("error_description", "Rate limit exceeded. Please try again later."),
                        new JProperty("retryAfter", 60),
                        new JProperty("requestId", context.RequestId)
                    ).ToString();
                }</set-body>
            </return-response>
        </when>
        <otherwise>
            <return-response>
                <set-status code="500" reason="Internal Server Error" />
                <set-header name="Content-Type" value="application/json" />
                <set-body>@{
                    return new JObject(
                        new JProperty("error", "internal_error"),
                        new JProperty("error_description", "An internal error occurred"),
                        new JProperty("requestId", context.RequestId)
                    ).ToString();
                }</set-body>
            </return-response>
        </otherwise>
    </choose>
    
    <trace source="GlobalErrorHandler"
           severity="error"
           message="@($"Error in operation {context.Operation?.Name}: {context.LastError?.Message}")">
        <metadata name="errorDetails" value="@((string)context.Variables["errorDetails"])" />
    </trace>
</on-error>
```

### Circuit Breaker Pattern
```xml
<!-- Implement circuit breaker for backend protection -->
<retry condition="@(context.Response.StatusCode >= 500)"
       count="3"
       interval="1"
       max-interval="10"
       delta="1">
    <forward-request timeout="30" />
</retry>
```

## Policy Fragments for Reusability

### Common Authentication Fragment
```xml
<!-- Policy Fragment: common-auth.xml -->
<fragment>
    <validate-azure-ad-token tenant-id="{{tenant-id}}">
        <audiences>
            <audience>{{api-audience}}</audience>
        </audiences>
        <required-claims>
            <claim name="scope" match="any">
                <value>api.read</value>
                <value>api.write</value>
            </claim>
        </required-claims>
    </validate-azure-ad-token>
    <set-variable name="userId" value="@(context.Request.Headers.GetValueOrDefault('Authorization','').AsJwt()?.Subject)" />
</fragment>
```

### Usage in Policies
```xml
<policies>
    <inbound>
        <base />
        <include-fragment fragment-id="common-auth" />
        <rate-limit-by-key calls="100"
                           renewal-period="60"
                           counter-key="@((string)context.Variables['userId'])" />
    </inbound>
</policies>
```

## Infrastructure as Code with Bicep

### Main Bicep Template (infra/main.bicep)
```bicep
targetScope = 'resourceGroup'

@description('Name of the existing API Management instance')
param apiManagementName string

@description('Environment name (dev, staging, prod)')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

@description('API configuration parameters')
param apiConfig object = {
  name: 'my-api'
  displayName: 'My API'
  description: 'Production API with comprehensive security'
  serviceUrl: 'https://backend.example.com/api'
  path: 'myapi'
  subscriptionRequired: true
  authenticationSettings: {
    oAuth2AuthenticationSettings: [
      {
        authorizationServerId: 'azure-ad-server'
        scope: 'api.read api.write'
      }
    ]
  }
}

// Reference existing API Management instance
resource apim 'Microsoft.ApiManagement/service@2023-05-01-preview' existing = {
  name: apiManagementName
}

// Deploy API configuration
module apiModule 'modules/apim/apis.bicep' = {
  name: 'api-deployment'
  params: {
    apiManagementName: apim.name
    apiConfig: apiConfig
    environment: environment
  }
}

// Deploy security policies
module policiesModule 'modules/apim/policies.bicep' = {
  name: 'policies-deployment'
  params: {
    apiManagementName: apim.name
    apiName: apiConfig.name
    environment: environment
  }
}

// Deploy backends
module backendsModule 'modules/apim/backends.bicep' = {
  name: 'backends-deployment'
  params: {
    apiManagementName: apim.name
    environment: environment
  }
}

// Deploy monitoring and logging
module monitoringModule 'modules/monitoring/logging.bicep' = {
  name: 'monitoring-deployment'
  params: {
    apiManagementName: apim.name
    environment: environment
  }
}
```

### API Module (infra/modules/apim/apis.bicep)
```bicep
@description('Name of the API Management instance')
param apiManagementName string

@description('API configuration')
param apiConfig object

@description('Environment name')
param environment string

resource apim 'Microsoft.ApiManagement/service@2023-05-01-preview' existing = {
  name: apiManagementName
}

// Create API
resource api 'Microsoft.ApiManagement/service/apis@2023-05-01-preview' = {
  parent: apim
  name: apiConfig.name
  properties: {
    displayName: apiConfig.displayName
    description: apiConfig.description
    serviceUrl: apiConfig.serviceUrl
    path: apiConfig.path
    protocols: ['https']
    subscriptionRequired: apiConfig.subscriptionRequired
    authenticationSettings: apiConfig.authenticationSettings
    subscriptionKeyParameterNames: {
      header: 'Ocp-Apim-Subscription-Key'
      query: 'subscription-key'
    }
    format: 'openapi+json'
    value: loadTextContent('../../src/apim/schemas/${apiConfig.name}-openapi.json')
  }
}

// Create API operations with individual policies
resource apiOperations 'Microsoft.ApiManagement/service/apis/operations@2023-05-01-preview' = [for operation in apiConfig.operations: {
  parent: api
  name: operation.name
  properties: {
    displayName: operation.displayName
    method: operation.method
    urlTemplate: operation.urlTemplate
    description: operation.description
    request: operation.request
    responses: operation.responses
    policies: {
      value: loadTextContent('../../src/apim/policies/apis/${apiConfig.name}/operations/${operation.name}/policy.xml')
    }
  }
}]

// Output API information
output apiId string = api.id
output apiName string = api.name
output apiPath string = api.properties.path
```

### Policies Module (infra/modules/apim/policies.bicep)
```bicep
@description('Name of the API Management instance')
param apiManagementName string

@description('API name')
param apiName string

@description('Environment name')
param environment string

resource apim 'Microsoft.ApiManagement/service@2023-05-01-preview' existing = {
  name: apiManagementName
}

resource api 'Microsoft.ApiManagement/service/apis@2023-05-01-preview' existing = {
  parent: apim
  name: apiName
}

// Deploy policy fragments
resource policyFragments 'Microsoft.ApiManagement/service/policyFragments@2023-05-01-preview' = [for fragment in ['common-auth', 'error-handling', 'rate-limiting']: {
  parent: apim
  name: fragment
  properties: {
    description: 'Reusable policy fragment: ${fragment}'
    value: loadTextContent('../../src/apim/policies/fragments/${fragment}.xml')
  }
}]

// Apply API-level policy
resource apiPolicy 'Microsoft.ApiManagement/service/apis/policies@2023-05-01-preview' = {
  parent: api
  name: 'policy'
  properties: {
    value: loadTextContent('../../src/apim/policies/apis/${apiName}/policy.xml')
    format: 'xml'
  }
}

// Apply global policy
resource globalPolicy 'Microsoft.ApiManagement/service/policies@2023-05-01-preview' = {
  parent: apim
  name: 'policy'
  properties: {
    value: loadTextContent('../../src/apim/policies/global/policy.xml')
    format: 'xml'
  }
}
```

### Backends Module (infra/modules/apim/backends.bicep)
```bicep
@description('Name of the API Management instance')
param apiManagementName string

@description('Environment name')
param environment string

resource apim 'Microsoft.ApiManagement/service@2023-05-01-preview' existing = {
  name: apiManagementName
}

// Create backend with managed identity authentication
resource backend 'Microsoft.ApiManagement/service/backends@2023-05-01-preview' = {
  parent: apim
  name: 'secure-backend'
  properties: {
    description: 'Backend service with managed identity auth'
    url: 'https://api-${environment}.example.com'
    protocol: 'http'
    circuitBreaker: {
      rules: [
        {
          failureCondition: {
            count: 5
            percentage: 50
            interval: 'PT30S'
            statusCodeRanges: [
              {
                min: 500
                max: 599
              }
            ]
          }
          tripDuration: 'PT60S'
        }
      ]
    }
    credentials: {
      authorization: {
        scheme: 'Bearer'
        parameter: '{{managed-identity-token}}'
      }
    }
    proxy: {
      url: 'https://proxy.example.com:8080'
      username: '{{proxy-username}}'
      password: '{{proxy-password}}'
    }
    tls: {
      validateCertificateChain: true
      validateCertificateName: true
    }
  }
}
```

## GitHub Actions CI/CD Pipeline

### Main Workflow (.github/workflows/apim-deploy.yml)
```yaml
name: Deploy to Azure API Management

on:
  push:
    branches: [main]
    paths: 
      - 'src/apim/**'
      - 'infra/**'
  pull_request:
    branches: [main]
    paths:
      - 'src/apim/**'
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
    name: Validate API Definitions and Policies
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js for API validation
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install API validation tools
        run: |
          npm install -g @apidevtools/swagger-cli
          npm install -g @stoplight/spectral-cli

      - name: Validate OpenAPI specifications
        run: |
          for spec in src/apim/schemas/*.json; do
            echo "Validating $spec"
            swagger-cli validate "$spec"
            spectral lint "$spec" --format pretty --verbose
          done

      - name: Validate XML policies
        run: |
          sudo apt-get update && sudo apt-get install -y libxml2-utils
          find src/apim/policies -name "*.xml" -exec xmllint --noout {} \;

      - name: Setup Bicep
        uses: Azure/bicep-build-action@v1.0.1
        with:
          bicepFilePath: infra/main.bicep

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
          skip_check: CKV_AZURE_115 # Skip specific checks if needed

      - name: Upload Checkov results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: reports/checkov-results.sarif

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
            apiManagementName=${{ vars.APIM_NAME }}
            environment=dev
          additionalArguments: --what-if --what-if-result-format FullResourcePayloads

      - name: Comment What-If Results
        uses: actions/github-script@v7
        with:
          script: |
            const output = `
            ## Infrastructure Changes Preview 🔍
            
            The Bicep what-if analysis has been completed. Please review the changes above.
            
            - ✅ **Validation**: API definitions and policies are valid
            - ✅ **Security**: Checkov security scan completed
            - 📋 **Changes**: Review the what-if output above
            
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
      url: https://${{ vars.APIM_NAME }}.azure-api.net
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure API Management
        uses: azure/arm-deploy@v1
        id: deploy
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP }}
          template: infra/main.bicep
          parameters: >
            apiManagementName=${{ vars.APIM_NAME }}
            environment=dev
          deploymentName: apim-deployment-${{ github.run_number }}

      - name: Test API Health
        run: |
          echo "Testing API health endpoints..."
          sleep 30  # Allow propagation
          
          # Test API health
          response=$(curl -s -w "\n%{http_code}" \
            -H "Ocp-Apim-Subscription-Key: ${{ secrets.APIM_SUBSCRIPTION_KEY }}" \
            "https://${{ vars.APIM_NAME }}.azure-api.net/myapi/health")
          
          http_code=$(echo "$response" | tail -n1)
          body=$(echo "$response" | head -n -1)
          
          if [[ $http_code -eq 200 ]]; then
            echo "✅ API health check passed"
            echo "Response: $body"
          else
            echo "❌ API health check failed with status $http_code"
            echo "Response: $body"
            exit 1
          fi

      - name: Run API Integration Tests
        run: |
          echo "Running integration tests..."
          # Add your API integration tests here
          # Example: newman run tests/api-tests.json

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://${{ vars.APIM_NAME_STAGING }}.azure-api.net
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Staging APIM
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP_STAGING }}
          template: infra/main.bicep
          parameters: >
            apiManagementName=${{ vars.APIM_NAME_STAGING }}
            environment=staging
          deploymentName: apim-staging-${{ github.run_number }}

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://${{ vars.APIM_NAME_PROD }}.azure-api.net
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Production APIM
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP_PROD }}
          template: infra/main.bicep
          parameters: >
            apiManagementName=${{ vars.APIM_NAME_PROD }}
            environment=prod
          deploymentName: apim-prod-${{ github.run_number }}

      - name: Post-deployment verification
        run: |
          echo "🚀 Production deployment completed!"
          echo "Performing post-deployment checks..."
          
          # Test critical endpoints
          curl -f -H "Ocp-Apim-Subscription-Key: ${{ secrets.APIM_SUBSCRIPTION_KEY_PROD }}" \
            "https://${{ vars.APIM_NAME_PROD }}.azure-api.net/myapi/health" || exit 1
          
          echo "✅ Production deployment verified successfully"

  notify:
    name: Notify Teams
    runs-on: ubuntu-latest
    needs: [deploy-prod]
    if: always()
    steps:
      - name: Notify on Success
        if: needs.deploy-prod.result == 'success'
        uses: 8398a7/action-slack@v3
        with:
          status: success
          fields: repo,commit,author,took
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      - name: Notify on Failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          fields: repo,commit,author,took
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## Additional Best Practices

### Named Values and Key Vault Integration
```bicep
// Use Key Vault references for sensitive values
resource namedValues 'Microsoft.ApiManagement/service/namedValues@2023-05-01-preview' = [for item in [
  { name: 'backend-api-key', displayName: 'Backend API Key', keyVault: true, secretId: '${keyVaultUri}secrets/backend-api-key' }
  { name: 'tenant-id', displayName: 'Azure AD Tenant ID', value: tenantId }
  { name: 'certificate-thumbprint', displayName: 'Client Certificate Thumbprint', keyVault: true, secretId: '${keyVaultUri}secrets/cert-thumbprint' }
]: {
  parent: apim
  name: item.name
  properties: {
    displayName: item.displayName
    value: item.keyVault ? null : item.value
    keyVault: item.keyVault ? {
      secretIdentifier: item.secretId
    } : null
    secret: item.keyVault
  }
}]
```

### API Versioning Strategy
```xml
<!-- Version-aware routing policy -->
<choose>
    <when condition="@(context.Request.Headers.GetValueOrDefault('Api-Version', '') == 'v2')">
        <set-backend-service base-url="https://api-v2.backend.com" />
    </when>
    <when condition="@(context.Request.Headers.GetValueOrDefault('Api-Version', '') == 'v1')">
        <set-backend-service base-url="https://api-v1.backend.com" />
    </when>
    <otherwise>
        <return-response>
            <set-status code="400" reason="Bad Request" />
            <set-header name="Content-Type" value="application/json" />
            <set-body>@{
                return new JObject(
                    new JProperty("error", "missing_version"),
                    new JProperty("error_description", "Api-Version header is required"),
                    new JProperty("supported_versions", new JArray("v1", "v2"))
                ).ToString();
            }</set-body>
        </return-response>
    </otherwise>
</choose>
```

### Health Check Implementation
```xml
<!-- Health check endpoint with dependency validation -->
<choose>
    <when condition="@(context.Request.Url.Path.EndsWith('/health'))">
        <send-request mode="new" response-variable-name="backendHealth" timeout="10" ignore-error="true">
            <set-url>@(context.Variables.GetValueOrDefault<string>("backend-url") + "/health")</set-url>
            <set-method>GET</set-method>
        </send-request>
        
        <return-response>
            <set-status code="@((IResponse)context.Variables['backendHealth']).StatusCode" reason="@((IResponse)context.Variables['backendHealth']).StatusReason" />
            <set-header name="Content-Type" value="application/json" />
            <set-body>@{
                var backendResponse = (IResponse)context.Variables["backendHealth"];
                var status = backendResponse.StatusCode == 200 ? "healthy" : "unhealthy";
                
                return new JObject(
                    new JProperty("status", status),
                    new JProperty("timestamp", DateTime.UtcNow),
                    new JProperty("version", "1.0.0"),
                    new JProperty("dependencies", new JObject(
                        new JProperty("backend", new JObject(
                            new JProperty("status", status),
                            new JProperty("responseTime", context.Elapsed.TotalMilliseconds)
                        ))
                    ))
                ).ToString();
            }</set-body>
        </return-response>
    </when>
</choose>
```

## Environment-Specific Configuration

### Development Environment
- Relaxed rate limits for testing
- Detailed error messages
- API tracing enabled (with caution)
- Subscription key requirements relaxed

### Staging Environment
- Production-like rate limits
- Reduced error detail exposure
- Performance testing enabled
- Full security validation

### Production Environment
- Strict rate limiting and quotas
- Minimal error detail exposure
- No API tracing
- Full security policies active
- Comprehensive monitoring

## Monitoring and Alerting

### Azure Monitor Integration
```bicep
// Application Insights for APIM
resource applicationInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'apim-insights-${environment}'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    Flow_Type: 'Redfield'
    Request_Source: 'IbizaAIExtension'
    WorkspaceResourceId: logAnalyticsWorkspace.id
  }
}

// APIM Logger for Application Insights
resource apimLogger 'Microsoft.ApiManagement/service/loggers@2023-05-01-preview' = {
  parent: apim
  name: 'applicationinsights'
  properties: {
    loggerType: 'applicationInsights'
    credentials: {
      instrumentationKey: applicationInsights.properties.InstrumentationKey
    }
    isBuffered: true
    resourceId: applicationInsights.id
  }
}

// Diagnostic settings for comprehensive logging
resource diagnosticSettings 'Microsoft.ApiManagement/service/diagnostics@2023-05-01-preview' = {
  parent: apim
  name: 'applicationinsights'
  properties: {
    alwaysLog: 'allErrors'
    loggerId: apimLogger.id
    sampling: {
      samplingType: 'fixed'
      percentage: 100
    }
    frontend: {
      request: {
        headers: ['Authorization']
        body: {
          bytes: 1024
        }
      }
      response: {
        headers: ['Content-Type']
        body: {
          bytes: 1024
        }
      }
    }
    backend: {
      request: {
        headers: ['User-Agent']
        body: {
          bytes: 1024
        }
      }
      response: {
        headers: ['Content-Type']
        body: {
          bytes: 1024
        }
      }
    }
    metrics: true
  }
}
```

## Additional References

### Official Microsoft Documentation
- [Azure API Management Documentation](https://learn.microsoft.com/en-us/azure/api-management/)
- [API Management Policies Reference](https://learn.microsoft.com/en-us/azure/api-management/api-management-policies)
- [API Management Policy Expressions](https://learn.microsoft.com/en-us/azure/api-management/api-management-policy-expressions)
- [OWASP API Security Top 10 with API Management](https://learn.microsoft.com/en-us/azure/api-management/mitigate-owasp-api-threats)
- [Azure API Management Best Practices](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-api-management)

### Advanced Topics
- [API Management with Virtual Networks](https://learn.microsoft.com/en-us/azure/api-management/virtual-network-concepts)
- [Managed Identity in API Management](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-managed-service-identity)
- [API Management DevOps with APIOps](https://github.com/Azure/APIOps)
- [Azure API Management Landing Zone Accelerator](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/app-platform/api-management/)

### Security and Compliance
- [API Authentication and Authorization](https://learn.microsoft.com/en-us/azure/api-management/authentication-authorization-overview)
- [Client Certificate Authentication](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-mutual-certificates-for-clients)
- [API Management Security Best Practices](https://learn.microsoft.com/en-us/azure/api-management/mitigate-owasp-api-threats)

### Monitoring and Observability
- [Monitor API Management](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor)
- [API Management Analytics](https://learn.microsoft.com/en-us/azure/api-management/howto-use-analytics)
- [Application Insights Integration](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-app-insights)

### Infrastructure as Code
- [Bicep Template for API Management](https://learn.microsoft.com/en-us/azure/templates/microsoft.apimanagement/service)
- [Deploy Bicep with GitHub Actions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-github-actions)
- [Bicep Best Practices](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/best-practices)

### Performance and Scaling
- [API Management Caching](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-cache)
- [Rate Limiting in API Management](https://learn.microsoft.com/en-us/azure/api-management/api-management-sample-flexible-throttling)
- [API Management Capacity Planning](https://learn.microsoft.com/en-us/azure/api-management/api-management-capacity)
