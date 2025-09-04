---
description: 'Best practices for developing, securing, monitoring, and deploying Azure Functions to existing App Service plans'
applyTo: "**/src/functions/**/*,**/infra/**/*"
---

# Azure Functions Development and Deployment Best Practices

## Overview

This guide provides comprehensive best practices for developing, securing, monitoring, and deploying **Azure Functions to existing App Service plans** using GitHub Actions and Infrastructure as Code (Bicep). Focus is on isolated process model, enterprise-grade security, and automated deployment patterns.

## Prerequisites

- Existing Azure App Service Plan (Premium v2/v3 or Elastic Premium recommended)
- GitHub repository with Actions enabled
- Azure service principal or workload identity for deployment
- Bicep CLI and Azure CLI installed for local development
- Azure Functions Core Tools v4.x for local development

## Project Structure

```
src/
├── functions/
│   ├── [FunctionApp]/
│   │   ├── host.json              # Function host configuration
│   │   ├── local.settings.json    # Development settings (not committed)
│   │   ├── Program.cs             # Application startup and DI
│   │   ├── Functions/
│   │   │   ├── [FunctionName].cs  # Individual function files
│   │   │   └── ...
│   │   ├── Models/                # Data models and DTOs
│   │   ├── Services/              # Business logic services
│   │   ├── Extensions/            # Extension methods and utilities
│   │   └── Configuration/         # Configuration classes
│   └── shared/
│       ├── Common/                # Shared utilities across function apps
│       └── Models/                # Shared data models
infra/
├── modules/
│   ├── functions/
│   │   ├── functionApp.bicep
│   │   ├── appSettings.bicep
│   │   └── monitoring.bicep
│   └── shared/
│       ├── keyVault.bicep
│       └── storage.bicep
└── main.bicep
```

## Azure Functions Specific Instructions

- Always use the latest Azure Functions .NET version and **isolated process model** over in-process for better performance and maintainability.
- Use **Functions Host v4** with the latest extension bundles version `[4.*, 5.0.0)` in host.json.
- Prefer **extension bundles over individual SDKs** for bindings to simplify dependency management.
- Always use the latest C# version, currently C# 13 features, with file-scoped namespaces.
- For **blob triggers**, always use EventGrid source for better performance and reliability.
- Set appropriate **authentication levels** (default: `function`) - avoid anonymous access in production.
- Write clear and concise comments for each function, especially explaining trigger configurations and business logic.

## Security Best Practices

### Authentication and Authorization

#### 1. Azure Active Directory Integration
```csharp
// Configure Azure AD authentication in Program.cs
var builder = FunctionsApplication.CreateBuilder(args);

builder.ConfigureFunctionsWebApplication(app =>
{
    app.UseAuthentication();
    app.UseAuthorization();
});

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

// Function with authentication required
[Function("SecureFunction")]
[Authorize]
public async Task<HttpResponseData> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequestData req,
    FunctionContext context)
{
    var user = context.GetHttpContext()?.User;
    if (user?.Identity?.IsAuthenticated != true)
    {
        var response = req.CreateResponse(HttpStatusCode.Unauthorized);
        return response;
    }
    
    // Function logic here
}
```

#### 2. Managed Identity for Azure Services
```csharp
// Configure Managed Identity in Program.cs
builder.Services.AddSingleton<TokenCredential>(provider =>
    new DefaultAzureCredential());

// Use Managed Identity for Azure services
public class StorageService
{
    private readonly BlobServiceClient _blobClient;
    
    public StorageService(IConfiguration configuration, TokenCredential credential)
    {
        var storageUri = configuration.GetValue<string>("Storage:BlobUri");
        _blobClient = new BlobServiceClient(new Uri(storageUri), credential);
    }
}
```

#### 3. Function-level Security
```csharp
// Implement custom authorization policies
[Function("AdminFunction")]
[Authorize(Policy = "RequireAdminRole")]
public async Task<HttpResponseData> AdminOnlyFunction(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req)
{
    // Admin-only functionality
}

// Configure policies in Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireAdminRole", policy =>
        policy.RequireClaim("roles", "FunctionAdmin"));
});
```

### Input Validation and Data Protection
```csharp
// Model validation with Data Annotations
public class CreateOrderRequest
{
    [Required]
    [StringLength(100, MinimumLength = 1)]
    public string ProductName { get; set; } = string.Empty;
    
    [Range(0.01, 10000.00)]
    public decimal Price { get; set; }
    
    [EmailAddress]
    public string CustomerEmail { get; set; } = string.Empty;
}

// Function with input validation
[Function("CreateOrder")]
public async Task<HttpResponseData> CreateOrder(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req,
    FunctionContext context)
{
    var request = await req.ReadFromJsonAsync<CreateOrderRequest>();
    if (request == null)
    {
        return req.CreateResponse(HttpStatusCode.BadRequest);
    }
    
    // Validate model
    var validationResults = new List<ValidationResult>();
    var isValid = Validator.TryValidateObject(
        request, new ValidationContext(request), validationResults, true);
    
    if (!isValid)
    {
        var response = req.CreateResponse(HttpStatusCode.BadRequest);
        await response.WriteAsJsonAsync(new { errors = validationResults });
        return response;
    }
    
    // Process valid request
}
```

## Configuration and Settings Management

### Secure Configuration with Key Vault
```csharp
// Program.cs - Key Vault integration
var builder = FunctionsApplication.CreateBuilder(args);

if (!builder.Environment.IsDevelopment())
{
    var keyVaultUri = builder.Configuration.GetValue<string>("KeyVault:VaultUri");
    if (!string.IsNullOrEmpty(keyVaultUri))
    {
        builder.Configuration.AddAzureKeyVault(
            new Uri(keyVaultUri),
            new DefaultAzureCredential());
    }
}

// Strongly typed configuration
public class DatabaseOptions
{
    public const string SectionName = "Database";
    
    [Required]
    public string ConnectionString { get; set; } = string.Empty;
    
    [Range(1, 300)]
    public int CommandTimeout { get; set; } = 30;
}

// Register configuration
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection(DatabaseOptions.SectionName));
```

### host.json Configuration
```json
{
  "version": "2.0",
  "functionTimeout": "00:05:00",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "maxTelemetryItemsPerSecond": 20,
        "excludedTypes": "Request;Exception"
      },
      "enableLiveMetricsFilters": true
    },
    "logLevel": {
      "default": "Information",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  },
  "healthMonitor": {
    "enabled": true,
    "healthCheckInterval": "00:00:10",
    "healthCheckWindow": "00:02:00",
    "healthCheckThreshold": 6,
    "counterThreshold": 0.80
  },
  "retry": {
    "strategy": "exponentialBackoff",
    "maxRetryCount": 3,
    "minimumInterval": "00:00:02",
    "maximumInterval": "00:00:30"
  },
  "concurrency": {
    "dynamicConcurrencyEnabled": true,
    "snapshotPersistenceEnabled": true
  }
}
```

## Dependency Injection and Service Configuration

```csharp
// Program.cs - Comprehensive service registration
var builder = FunctionsApplication.CreateBuilder(args);

builder.ConfigureFunctionsWebApplication(app =>
{
    app.UseAuthentication();
    app.UseAuthorization();
    app.UseMiddleware<CorrelationIdMiddleware>();
    app.UseMiddleware<ExceptionHandlingMiddleware>();
});

// Core services
builder.Services.AddApplicationInsightsTelemetryWorkerService();
builder.Services.ConfigureFunctionsApplicationInsights();

// HTTP clients with resilience
builder.Services.AddHttpClient<IExternalApiService, ExternalApiService>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration.GetValue<string>("ExternalApi:BaseUrl")!);
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

// Database services
builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));
});

// Custom services
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<INotificationService, NotificationService>();
builder.Services.AddSingleton<IMemoryCache, MemoryCache>();

// Configuration options
builder.Services.Configure<ExternalApiOptions>(
    builder.Configuration.GetSection("ExternalApi"));

var app = builder.Build();
app.Run();

// Resilience policies
static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
            onRetry: (outcome, timespan, retryCount, context) =>
            {
                Console.WriteLine($"Retry {retryCount} after {timespan} seconds");
            });
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 3,
            durationOfBreak: TimeSpan.FromSeconds(30));
}
```

## Azure Functions Triggers and Bindings

### HTTP Triggers with Advanced Configuration
```csharp
[Function("ProcessOrder")]
public async Task<HttpResponseData> ProcessOrder(
    [HttpTrigger(
        AuthorizationLevel.Function, 
        "post", 
        Route = "orders/{orderId:guid}")] HttpRequestData req,
    [FromRoute] Guid orderId,
    FunctionContext context)
{
    var logger = context.GetLogger<OrderFunctions>();
    
    using var activity = ActivitySource.StartActivity("ProcessOrder");
    activity?.SetTag("orderId", orderId.ToString());
    
    try
    {
        // Function logic
        var response = req.CreateResponse(HttpStatusCode.OK);
        await response.WriteAsJsonAsync(new { orderId, status = "processed" });
        return response;
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Error processing order {OrderId}", orderId);
        return req.CreateResponse(HttpStatusCode.InternalServerError);
    }
}
```

### Queue Triggers with Poison Message Handling
```csharp
[Function("ProcessQueueMessage")]
public async Task ProcessQueueMessage(
    [QueueTrigger("orders", Connection = "AzureWebJobsStorage")] QueueMessage message,
    FunctionContext context)
{
    var logger = context.GetLogger<QueueProcessor>();
    
    try
    {
        var orderData = JsonSerializer.Deserialize<OrderData>(message.MessageText);
        if (orderData == null)
        {
            logger.LogWarning("Invalid message format: {MessageId}", message.MessageId);
            return; // Message will be deleted
        }
        
        await ProcessOrderAsync(orderData);
        logger.LogInformation("Successfully processed order {OrderId}", orderData.OrderId);
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Error processing queue message {MessageId}", message.MessageId);
        throw; // Trigger retry mechanism
    }
}
```

### Timer Triggers with Timezone Support
```csharp
[Function("DailyReport")]
public async Task GenerateDailyReport(
    [TimerTrigger("0 0 8 * * *", TimeZoneInfo = "Eastern Standard Time")] TimerInfo timer,
    FunctionContext context)
{
    var logger = context.GetLogger<ReportFunctions>();
    
    if (timer.IsPastDue)
    {
        logger.LogInformation("Timer is past due, but proceeding with report generation");
    }
    
    logger.LogInformation("Generating daily report at {ScheduleStatus}", timer.ScheduleStatus?.ToString());
    
    // Report generation logic
    await GenerateReportAsync();
}
```

### Event Grid Triggers for Better Performance
```csharp
[Function("ProcessBlobEvent")]
public async Task ProcessBlobCreated(
    [EventGridTrigger] EventGridEvent eventGridEvent,
    FunctionContext context)
{
    var logger = context.GetLogger<BlobProcessor>();
    
    if (eventGridEvent.EventType == "Microsoft.Storage.BlobCreated")
    {
        var blobData = eventGridEvent.Data?.ToObjectFromJson<StorageBlobCreatedEventData>();
        if (blobData != null)
        {
            logger.LogInformation("Processing blob: {BlobUrl}", blobData.Url);
            await ProcessBlobAsync(blobData.Url);
        }
    }
}
```

## Error Handling and Resilience

### Global Exception Handling Middleware
```csharp
public class ExceptionHandlingMiddleware : IFunctionsWorkerMiddleware
{
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    
    public ExceptionHandlingMiddleware(ILogger<ExceptionHandlingMiddleware> logger)
    {
        _logger = logger;
    }
    
    public async Task Invoke(FunctionContext context, FunctionExecutionDelegate next)
    {
        try
        {
            await next(context);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Validation error in function {FunctionName}", context.FunctionDefinition.Name);
            await HandleValidationExceptionAsync(context, ex);
        }
        catch (UnauthorizedAccessException ex)
        {
            _logger.LogWarning(ex, "Unauthorized access in function {FunctionName}", context.FunctionDefinition.Name);
            await HandleUnauthorizedExceptionAsync(context, ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception in function {FunctionName}", context.FunctionDefinition.Name);
            await HandleGenericExceptionAsync(context, ex);
        }
    }
    
    private async Task HandleValidationExceptionAsync(FunctionContext context, ValidationException ex)
    {
        var httpContext = context.GetHttpContext();
        if (httpContext?.Response != null)
        {
            httpContext.Response.StatusCode = 400;
            await httpContext.Response.WriteAsJsonAsync(new
            {
                error = "validation_failed",
                message = ex.Message,
                details = ex.ValidationResult
            });
        }
    }
}
```

### Structured Logging with Correlation IDs
```csharp
public class CorrelationIdMiddleware : IFunctionsWorkerMiddleware
{
    public async Task Invoke(FunctionContext context, FunctionExecutionDelegate next)
    {
        var correlationId = Guid.NewGuid().ToString();
        
        // Add correlation ID to logging scope
        using var scope = context.GetLogger<CorrelationIdMiddleware>()
            .BeginScope(new Dictionary<string, object> { ["CorrelationId"] = correlationId });
        
        // Add correlation ID to HTTP response if applicable
        var httpContext = context.GetHttpContext();
        if (httpContext?.Response != null)
        {
            httpContext.Response.Headers.Add("X-Correlation-ID", correlationId);
        }
        
        await next(context);
    }
}
```

## Performance Optimization

### Connection Management and Pooling
```csharp
// HTTP Client service with connection pooling
public class ExternalApiService : IExternalApiService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ExternalApiService> _logger;
    
    public ExternalApiService(HttpClient httpClient, ILogger<ExternalApiService> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }
    
    public async Task<ApiResponse<T>> GetAsync<T>(string endpoint)
    {
        using var activity = ActivitySource.StartActivity("ExternalApiCall");
        activity?.SetTag("endpoint", endpoint);
        
        try
        {
            var response = await _httpClient.GetAsync(endpoint);
            response.EnsureSuccessStatusCode();
            
            var content = await response.Content.ReadAsStringAsync();
            var result = JsonSerializer.Deserialize<T>(content);
            
            return new ApiResponse<T> { Data = result, IsSuccess = true };
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "HTTP error calling {Endpoint}", endpoint);
            return new ApiResponse<T> { IsSuccess = false, Error = ex.Message };
        }
    }
}
```

### Efficient Memory Usage
```csharp
[Function("ProcessLargeFile")]
public async Task ProcessLargeFile(
    [BlobTrigger("input/{name}")] Stream blobStream,
    string name,
    [Blob("output/{name}.processed")] Stream outputStream,
    FunctionContext context)
{
    var logger = context.GetLogger<FileProcessor>();
    
    // Use streaming for large files to minimize memory usage
    using var reader = new StreamReader(blobStream);
    using var writer = new StreamWriter(outputStream);
    
    string? line;
    var lineCount = 0;
    
    while ((line = await reader.ReadLineAsync()) != null)
    {
        var processedLine = ProcessLine(line);
        await writer.WriteLineAsync(processedLine);
        
        lineCount++;
        if (lineCount % 1000 == 0)
        {
            logger.LogInformation("Processed {LineCount} lines from {FileName}", lineCount, name);
        }
    }
    
    logger.LogInformation("Completed processing {TotalLines} lines from {FileName}", lineCount, name);
}
```

## Infrastructure as Code with Bicep

### Main Bicep Template (infra/main.bicep)
```bicep
targetScope = 'resourceGroup'

@description('Name of the existing App Service Plan')
param appServicePlanName string

@description('Function App name')
@minLength(2)
@maxLength(60)
param functionAppName string

@description('Environment name (dev, staging, prod)')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

@description('Application insights name')
param applicationInsightsName string = '${functionAppName}-insights'

@description('Storage account name for function app')
@minLength(3)
@maxLength(24)
param storageAccountName string

@description('Key Vault name for secrets')
param keyVaultName string

@description('Function app configuration')
param functionAppConfig object = {
  runtime: 'dotnet-isolated'
  version: '8.0'
  alwaysOn: true
  use32BitWorkerProcess: false
  ftpsState: 'Disabled'
  minTlsVersion: '1.2'
  scmMinTlsVersion: '1.2'
  httpsOnly: true
}

// Reference existing App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' existing = {
  name: appServicePlanName
}

// Deploy Function App
module functionApp 'modules/functions/functionApp.bicep' = {
  name: 'functionApp-deployment'
  params: {
    functionAppName: functionAppName
    appServicePlanId: appServicePlan.id
    storageAccountName: storageAccountName
    applicationInsightsName: applicationInsightsName
    keyVaultName: keyVaultName
    environment: environment
    functionAppConfig: functionAppConfig
  }
}

// Deploy monitoring and logging
module monitoring 'modules/functions/monitoring.bicep' = {
  name: 'monitoring-deployment'
  params: {
    functionAppName: functionAppName
    applicationInsightsName: applicationInsightsName
    environment: environment
  }
}

// Output important information
output functionAppName string = functionApp.outputs.functionAppName
output functionAppUrl string = functionApp.outputs.functionAppUrl
output applicationInsightsInstrumentationKey string = monitoring.outputs.instrumentationKey
```

### Function App Module (infra/modules/functions/functionApp.bicep)
```bicep
@description('Function App name')
param functionAppName string

@description('App Service Plan resource ID')
param appServicePlanId string

@description('Storage account name')
param storageAccountName string

@description('Application Insights name')
param applicationInsightsName string

@description('Key Vault name')
param keyVaultName string

@description('Environment name')
param environment string

@description('Function app configuration')
param functionAppConfig object

// Create storage account for Function App
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    networkAcls: {
      defaultAction: 'Allow'
    }
    encryption: {
      services: {
        blob: {
          enabled: true
        }
        file: {
          enabled: true
        }
      }
      keySource: 'Microsoft.Storage'
    }
  }
}

// Reference existing Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: keyVaultName
}

// Reference Application Insights
resource applicationInsights 'Microsoft.Insights/components@2020-02-02' existing = {
  name: applicationInsightsName
}

// Create Function App
resource functionApp 'Microsoft.Web/sites@2023-01-01' = {
  name: functionAppName
  location: resourceGroup().location
  kind: 'functionapp'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlanId
    httpsOnly: functionAppConfig.httpsOnly
    siteConfig: {
      netFrameworkVersion: 'v8.0'
      use32BitWorkerProcess: functionAppConfig.use32BitWorkerProcess
      alwaysOn: functionAppConfig.alwaysOn
      ftpsState: functionAppConfig.ftpsState
      minTlsVersion: functionAppConfig.minTlsVersion
      scmMinTlsVersion: functionAppConfig.scmMinTlsVersion
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
          value: toLower(functionAppName)
        }
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
        }
        {
          name: 'WEBSITE_NODE_DEFAULT_VERSION'
          value: '~18'
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: 'dotnet-isolated'
        }
        {
          name: 'APPINSIGHTS_INSTRUMENTATIONKEY'
          value: applicationInsights.properties.InstrumentationKey
        }
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: applicationInsights.properties.ConnectionString
        }
        {
          name: 'ENVIRONMENT'
          value: environment
        }
        {
          name: 'KeyVault:VaultUri'
          value: keyVault.properties.vaultUri
        }
      ]
      connectionStrings: [
        {
          name: 'DefaultConnection'
          connectionString: '@Microsoft.KeyVault(VaultName=${keyVault.name};SecretName=DefaultConnection)'
          type: 'SQLAzure'
        }
      ]
      cors: {
        allowedOrigins: [
          'https://portal.azure.com'
        ]
        supportCredentials: false
      }
    }
  }
}

// Grant Function App access to Key Vault
resource keyVaultAccessPolicy 'Microsoft.KeyVault/vaults/accessPolicies@2023-07-01' = {
  parent: keyVault
  name: 'add'
  properties: {
    accessPolicies: [
      {
        tenantId: functionApp.identity.tenantId
        objectId: functionApp.identity.principalId
        permissions: {
          secrets: [
            'get'
            'list'
          ]
        }
      }
    ]
  }
}

// Function App deployment slots for zero-downtime deployments
resource functionAppStagingSlot 'Microsoft.Web/sites/slots@2023-01-01' = if (environment == 'prod') {
  parent: functionApp
  name: 'staging'
  location: resourceGroup().location
  kind: 'functionapp'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlanId
    httpsOnly: functionAppConfig.httpsOnly
    siteConfig: functionApp.properties.siteConfig
  }
}

output functionAppName string = functionApp.name
output functionAppUrl string = 'https://${functionApp.properties.defaultHostName}'
output functionAppPrincipalId string = functionApp.identity.principalId
output functionAppResourceId string = functionApp.id
```

### Monitoring Module (infra/modules/functions/monitoring.bicep)
```bicep
@description('Function App name')
param functionAppName string

@description('Application Insights name')
param applicationInsightsName string

@description('Environment name')
param environment string

// Create Log Analytics Workspace
resource logAnalyticsWorkspace 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: '${functionAppName}-logs'
  location: resourceGroup().location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: environment == 'prod' ? 90 : 30
    workspaceCapping: {
      dailyQuotaGb: environment == 'prod' ? 10 : 5
    }
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
}

// Create Application Insights
resource applicationInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: applicationInsightsName
  location: resourceGroup().location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    Flow_Type: 'Redfield'
    Request_Source: 'IbizaAIExtension'
    WorkspaceResourceId: logAnalyticsWorkspace.id
    SamplingPercentage: environment == 'prod' ? 10 : 100
  }
}

// Reference Function App
resource functionApp 'Microsoft.Web/sites@2023-01-01' existing = {
  name: functionAppName
}

// Create diagnostic settings for Function App
resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  scope: functionApp
  name: 'function-diagnostics'
  properties: {
    workspaceId: logAnalyticsWorkspace.id
    logs: [
      {
        category: 'FunctionAppLogs'
        enabled: true
        retentionPolicy: {
          days: environment == 'prod' ? 30 : 7
          enabled: true
        }
      }
    ]
    metrics: [
      {
        category: 'AllMetrics'
        enabled: true
        retentionPolicy: {
          days: environment == 'prod' ? 30 : 7
          enabled: true
        }
      }
    ]
  }
}

// Create alerts for critical metrics
resource functionFailureAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: '${functionAppName}-failure-rate'
  location: 'global'
  properties: {
    description: 'Alert when function failure rate exceeds threshold'
    severity: 2
    enabled: true
    scopes: [
      functionApp.id
    ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'Failure Rate'
          metricName: 'FunctionExecutionCount'
          dimensions: [
            {
              name: 'Status'
              operator: 'Include'
              values: [
                'Failed'
              ]
            }
          ]
          operator: 'GreaterThan'
          threshold: 5
          timeAggregation: 'Total'
        }
      ]
    }
    actions: [
      {
        actionGroupId: alertActionGroup.id
      }
    ]
  }
}

// Create Action Group for alerts
resource alertActionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: '${functionAppName}-alerts'
  location: 'global'
  properties: {
    groupShortName: 'FuncAlerts'
    enabled: true
    emailReceivers: [
      {
        name: 'DevTeam'
        emailAddress: 'dev-team@company.com'
        useCommonAlertSchema: true
      }
    ]
    smsReceivers: []
    webhookReceivers: []
    eventHubReceivers: []
    itsmReceivers: []
    azureAppPushReceivers: []
    automationRunbookReceivers: []
    voiceReceivers: []
    logicAppReceivers: []
    azureFunctionReceivers: []
    armRoleReceivers: []
  }
}

output instrumentationKey string = applicationInsights.properties.InstrumentationKey
output connectionString string = applicationInsights.properties.ConnectionString
output workspaceId string = logAnalyticsWorkspace.id
```

## GitHub Actions CI/CD Pipeline

### Main Workflow (.github/workflows/functions-deploy.yml)
```yaml
name: Deploy Azure Functions

on:
  push:
    branches: [main]
    paths: 
      - 'src/functions/**'
      - 'infra/**'
  pull_request:
    branches: [main]
    paths:
      - 'src/functions/**'
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

env:
  DOTNET_VERSION: '8.0.x'
  AZURE_FUNCTIONAPP_PACKAGE_PATH: './src/functions'

jobs:
  build-test:
    name: Build and Test Function Apps
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore dependencies
        run: |
          cd ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}
          dotnet restore

      - name: Build solution
        run: |
          cd ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}
          dotnet build --configuration Release --no-restore

      - name: Run unit tests
        run: |
          cd ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}
          dotnet test --configuration Release --no-build --verbosity normal --collect:"XPlat Code Coverage" --results-directory ./TestResults

      - name: Code Coverage Report
        uses: irongut/CodeCoverageSummary@v1.3.0
        with:
          filename: ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}/TestResults/**/coverage.cobertura.xml
          badge: true
          fail_below_min: true
          format: markdown
          hide_branch_rate: false
          hide_complexity: true
          indicators: true
          output: both
          thresholds: '60 80'

      - name: Security scan with CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: csharp

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

      - name: Publish Function App
        run: |
          cd ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}
          dotnet publish --configuration Release --output ./publish --no-build

      - name: Upload Function App artifacts
        uses: actions/upload-artifact@v4
        with:
          name: function-app-${{ github.sha }}
          path: ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}/publish

  validate-infrastructure:
    name: Validate Infrastructure
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Bicep
        uses: Azure/bicep-build-action@v1.0.1
        with:
          bicepFilePath: infra/main.bicep

      - name: Run Bicep Linter
        run: |
          az bicep lint --file infra/main.bicep

      - name: Security scan with Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: infra
          framework: bicep
          output_format: sarif
          output_file_path: reports/checkov-results.sarif

      - name: Upload security scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: reports/checkov-results.sarif

  plan-infrastructure:
    name: Plan Infrastructure Changes
    runs-on: ubuntu-latest
    needs: [build-test, validate-infrastructure]
    if: github.event_name == 'pull_request'
    environment: dev
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
            functionAppName=${{ vars.FUNCTION_APP_NAME }}-dev
            environment=dev
            storageAccountName=${{ vars.STORAGE_ACCOUNT_NAME }}dev
            keyVaultName=${{ vars.KEY_VAULT_NAME }}
          additionalArguments: --what-if --what-if-result-format FullResourcePayloads

  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: [build-test, validate-infrastructure]
    if: github.ref == 'refs/heads/main'
    environment:
      name: dev
      url: https://${{ vars.FUNCTION_APP_NAME }}-dev.azurewebsites.net
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: function-app-${{ github.sha }}
          path: ./publish

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Infrastructure
        uses: azure/arm-deploy@v1
        id: deploy-infra
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP }}
          template: infra/main.bicep
          parameters: >
            appServicePlanName=${{ vars.APP_SERVICE_PLAN_NAME }}
            functionAppName=${{ vars.FUNCTION_APP_NAME }}-dev
            environment=dev
            storageAccountName=${{ vars.STORAGE_ACCOUNT_NAME }}dev
            keyVaultName=${{ vars.KEY_VAULT_NAME }}
          deploymentName: functions-infra-${{ github.run_number }}

      - name: Deploy Function App
        uses: Azure/functions-action@v1
        with:
          app-name: ${{ vars.FUNCTION_APP_NAME }}-dev
          package: ./publish
          respect-funcignore: true

      - name: Wait for deployment
        run: sleep 60

      - name: Test Function App Health
        run: |
          echo "Testing Function App health..."
          
          # Get function key from Azure
          FUNCTION_KEY=$(az functionapp keys list \
            --name ${{ vars.FUNCTION_APP_NAME }}-dev \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --query "functionKeys.default" --output tsv)
          
          # Test health endpoint
          response=$(curl -s -w "\n%{http_code}" \
            -H "x-functions-key: $FUNCTION_KEY" \
            "https://${{ vars.FUNCTION_APP_NAME }}-dev.azurewebsites.net/api/health")
          
          http_code=$(echo "$response" | tail -n1)
          body=$(echo "$response" | head -n -1)
          
          if [[ $http_code -eq 200 ]]; then
            echo "✅ Function App health check passed"
            echo "Response: $body"
          else
            echo "❌ Function App health check failed with status $http_code"
            echo "Response: $body"
            exit 1
          fi

      - name: Run integration tests
        run: |
          echo "Running integration tests against dev environment..."
          # Add your integration test commands here

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://${{ vars.FUNCTION_APP_NAME }}-staging.azurewebsites.net
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: function-app-${{ github.sha }}
          path: ./publish

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Infrastructure
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP_STAGING }}
          template: infra/main.bicep
          parameters: >
            appServicePlanName=${{ vars.APP_SERVICE_PLAN_NAME_STAGING }}
            functionAppName=${{ vars.FUNCTION_APP_NAME }}-staging
            environment=staging
            storageAccountName=${{ vars.STORAGE_ACCOUNT_NAME }}stg
            keyVaultName=${{ vars.KEY_VAULT_NAME_STAGING }}

      - name: Deploy Function App
        uses: Azure/functions-action@v1
        with:
          app-name: ${{ vars.FUNCTION_APP_NAME }}-staging
          package: ./publish
          respect-funcignore: true

      - name: Run performance tests
        run: |
          echo "Running performance tests..."
          # Add performance test commands here

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://${{ vars.FUNCTION_APP_NAME }}.azurewebsites.net
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: function-app-${{ github.sha }}
          path: ./publish

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Infrastructure
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP_PROD }}
          template: infra/main.bicep
          parameters: >
            appServicePlanName=${{ vars.APP_SERVICE_PLAN_NAME_PROD }}
            functionAppName=${{ vars.FUNCTION_APP_NAME }}
            environment=prod
            storageAccountName=${{ vars.STORAGE_ACCOUNT_NAME }}prod
            keyVaultName=${{ vars.KEY_VAULT_NAME_PROD }}

      - name: Deploy to Staging Slot
        uses: Azure/functions-action@v1
        with:
          app-name: ${{ vars.FUNCTION_APP_NAME }}
          slot-name: staging
          package: ./publish
          respect-funcignore: true

      - name: Run smoke tests on staging slot
        run: |
          echo "Running smoke tests on staging slot..."
          
          FUNCTION_KEY=$(az functionapp keys list \
            --name ${{ vars.FUNCTION_APP_NAME }} \
            --slot staging \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP_PROD }} \
            --query "functionKeys.default" --output tsv)
          
          # Test critical endpoints
          curl -f -H "x-functions-key: $FUNCTION_KEY" \
            "https://${{ vars.FUNCTION_APP_NAME }}-staging.azurewebsites.net/api/health" || exit 1

      - name: Swap to Production
        run: |
          echo "Swapping staging slot to production..."
          az functionapp deployment slot swap \
            --name ${{ vars.FUNCTION_APP_NAME }} \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP_PROD }} \
            --slot staging \
            --target-slot production

      - name: Verify production deployment
        run: |
          echo "🚀 Production deployment completed!"
          echo "Verifying production deployment..."
          
          sleep 30  # Allow DNS propagation
          
          FUNCTION_KEY=$(az functionapp keys list \
            --name ${{ vars.FUNCTION_APP_NAME }} \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP_PROD }} \
            --query "functionKeys.default" --output tsv)
          
          curl -f -H "x-functions-key: $FUNCTION_KEY" \
            "https://${{ vars.FUNCTION_APP_NAME }}.azurewebsites.net/api/health" || exit 1
          
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
          message: '🚀 Azure Functions deployment completed successfully!'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      - name: Notify on Failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          fields: repo,commit,author,took
          message: '❌ Azure Functions deployment failed!'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## Testing Azure Functions

### Unit Testing with TestHost
```csharp
// Test setup
public class FunctionTests
{
    private readonly IHost _host;
    private readonly IServiceProvider _serviceProvider;

    public FunctionTests()
    {
        var builder = FunctionsApplication.CreateBuilder();
        builder.ConfigureFunctionsWebApplication();
        
        // Configure test services
        builder.Services.AddSingleton<IOrderService, MockOrderService>();
        
        _host = builder.Build();
        _serviceProvider = _host.Services;
    }

    [Fact]
    public async Task ProcessOrder_ValidRequest_ReturnsSuccess()
    {
        // Arrange
        var function = _serviceProvider.GetRequiredService<OrderFunctions>();
        var context = CreateFunctionContext();
        var request = CreateHttpRequestData();
        
        // Act
        var response = await function.ProcessOrder(request, Guid.NewGuid(), context);
        
        // Assert
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }

    private FunctionContext CreateFunctionContext()
    {
        var context = Substitute.For<FunctionContext>();
        var logger = Substitute.For<ILogger<OrderFunctions>>();
        context.GetLogger<OrderFunctions>().Returns(logger);
        return context;
    }
}
```

### Integration Testing
```csharp
public class IntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;

    public IntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureAppConfiguration(config =>
            {
                config.AddInMemoryCollection(new Dictionary<string, string?>
                {
                    ["Database:ConnectionString"] = "Server=(localdb)\\mssqllocaldb;Database=TestDb;Trusted_Connection=true",
                    ["ExternalApi:BaseUrl"] = "https://mockapi.example.com"
                });
            });
        });
        _client = _factory.CreateClient();
    }

    [Fact]
    public async Task HealthCheck_ReturnsOk()
    {
        // Act
        var response = await _client.GetAsync("/api/health");
        
        // Assert
        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        Assert.Contains("healthy", content);
    }
}
```

## Monitoring and Observability

### Custom Telemetry and Metrics
```csharp
public class TelemetryService
{
    private readonly TelemetryClient _telemetryClient;
    private readonly ILogger<TelemetryService> _logger;

    public TelemetryService(TelemetryClient telemetryClient, ILogger<TelemetryService> logger)
    {
        _telemetryClient = telemetryClient;
        _logger = logger;
    }

    public void TrackCustomEvent(string eventName, IDictionary<string, string> properties, IDictionary<string, double> metrics)
    {
        _telemetryClient.TrackEvent(eventName, properties, metrics);
        _logger.LogInformation("Custom event tracked: {EventName}", eventName);
    }

    public void TrackDependency(string dependencyType, string dependencyName, string data, DateTimeOffset startTime, TimeSpan duration, bool success)
    {
        _telemetryClient.TrackDependency(dependencyType, dependencyName, data, startTime, duration, success);
    }

    public void TrackBusinessMetric(string metricName, double value, IDictionary<string, string> properties = null)
    {
        _telemetryClient.GetMetric(metricName).TrackValue(value, properties);
        _logger.LogInformation("Business metric tracked: {MetricName} = {Value}", metricName, value);
    }
}
```

### Health Check Implementation
```csharp
[Function("HealthCheck")]
public async Task<HttpResponseData> HealthCheck(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "health")] HttpRequestData req,
    FunctionContext context)
{
    var logger = context.GetLogger<HealthFunctions>();
    
    var healthChecks = new Dictionary<string, object>();
    var overallStatus = "healthy";
    
    try
    {
        // Check database connectivity
        using var dbContext = context.InstanceServices.GetRequiredService<ApplicationDbContext>();
        await dbContext.Database.CanConnectAsync();
        healthChecks["database"] = new { status = "healthy", responseTime = "fast" };
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Database health check failed");
        healthChecks["database"] = new { status = "unhealthy", error = ex.Message };
        overallStatus = "unhealthy";
    }
    
    try
    {
        // Check external service connectivity
        var httpClient = context.InstanceServices.GetRequiredService<HttpClient>();
        using var response = await httpClient.GetAsync("https://api.external.com/health");
        response.EnsureSuccessStatusCode();
        healthChecks["externalService"] = new { status = "healthy" };
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "External service health check failed");
        healthChecks["externalService"] = new { status = "unhealthy", error = ex.Message };
        overallStatus = "degraded";
    }
    
    var healthResponse = new
    {
        status = overallStatus,
        timestamp = DateTime.UtcNow,
        version = "1.0.0",
        checks = healthChecks,
        environment = Environment.GetEnvironmentVariable("ENVIRONMENT") ?? "unknown"
    };
    
    var response = req.CreateResponse();
    response.StatusCode = overallStatus == "healthy" ? HttpStatusCode.OK : HttpStatusCode.ServiceUnavailable;
    await response.WriteAsJsonAsync(healthResponse);
    
    return response;
}
```

## Environment-Specific Configuration

### Development Environment
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "ENVIRONMENT": "Development",
    "Database:ConnectionString": "Server=(localdb)\\mssqllocaldb;Database=FunctionAppDev;Trusted_Connection=true",
    "ExternalApi:BaseUrl": "https://dev-api.example.com",
    "ExternalApi:ApiKey": "dev-api-key",
    "KeyVault:VaultUri": "",
    "ApplicationInsights:InstrumentationKey": ""
  },
  "Host": {
    "LocalHttpPort": 7071,
    "CORS": "*",
    "CORSCredentials": false
  }
}
```

### Production Configuration (from Key Vault)
- Database connection strings stored in Key Vault
- API keys and secrets managed through Managed Identity
- Application Insights configured for production sampling
- Enhanced security headers and CORS policies
- Comprehensive logging and monitoring enabled

## Additional Best Practices

### Connection Management
```csharp
// Proper HTTP client usage with dependency injection
public class ExternalApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ExternalApiClient> _logger;
    
    public ExternalApiClient(HttpClient httpClient, ILogger<ExternalApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }
    
    public async Task<T> GetAsync<T>(string endpoint)
    {
        using var activity = ActivitySource.StartActivity("ExternalApiCall");
        activity?.SetTag("endpoint", endpoint);
        
        try
        {
            var response = await _httpClient.GetAsync(endpoint);
            response.EnsureSuccessStatusCode();
            
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<T>(json, new JsonSerializerOptions
            {
                PropertyNameCaseInsensitive = true
            })!;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error calling external API endpoint: {Endpoint}", endpoint);
            throw;
        }
    }
}
```

### Async Best Practices
```csharp
[Function("ProcessItems")]
public async Task ProcessItems(
    [QueueTrigger("items")] QueueMessage message,
    FunctionContext context)
{
    var items = JsonSerializer.Deserialize<List<Item>>(message.MessageText);
    if (items == null) return;
    
    // Process items in parallel with controlled concurrency
    var semaphore = new SemaphoreSlim(10); // Limit to 10 concurrent operations
    var tasks = items.Select(async item =>
    {
        await semaphore.WaitAsync();
        try
        {
            await ProcessItemAsync(item);
        }
        finally
        {
            semaphore.Release();
        }
    });
    
    await Task.WhenAll(tasks);
}
```

### Memory Optimization
```csharp
[Function("ProcessLargeDataset")]
public async Task ProcessLargeDataset(
    [BlobTrigger("input/{name}")] Stream input,
    [Blob("output/{name}.processed")] Stream output,
    FunctionContext context)
{
    // Use streaming to handle large datasets without loading everything into memory
    using var reader = new JsonTextReader(new StreamReader(input));
    using var writer = new JsonTextWriter(new StreamWriter(output));
    
    var serializer = JsonSerializer.Create();
    
    reader.SupportMultipleContent = true;
    
    while (await reader.ReadAsync())
    {
        if (reader.TokenType == JsonToken.StartObject)
        {
            var item = serializer.Deserialize<DataItem>(reader);
            if (item != null)
            {
                var processedItem = await ProcessDataItemAsync(item);
                serializer.Serialize(writer, processedItem);
                await writer.FlushAsync();
            }
        }
    }
}
```

## Naming Conventions

- Follow PascalCase for **function names, method names, and public members**.
- Use camelCase for **private fields and local variables**.
- Prefix interface names with "I" (e.g., IUserService).
- Use **descriptive function names** that indicate the trigger type (e.g., `ProcessOrderHttpTrigger`, `SendEmailTimerTrigger`).
- Follow **Azure Functions naming conventions** for consistent identification in logs and monitoring.

## Formatting

- Apply code-formatting style defined in `.editorconfig`.
- Prefer file-scoped namespace declarations and single-line using directives.
- Insert a newline before the opening curly brace of any code block (e.g., after `if`, `for`, `while`, `foreach`, `using`, `try`, etc.).
- Ensure that the final return statement of a method is on its own line.
- Use pattern matching and switch expressions wherever possible.
- Use `nameof` instead of string literals when referring to member names.
- Ensure that XML doc comments are created for any public APIs. When applicable, include `<example>` and `<code>` documentation in the comments.

## Nullable Reference Types

- Declare variables non-nullable, and check for `null` at entry points.
- Always use `is null` or `is not null` instead of `== null` or `!= null`.
- Trust the C# null annotations and don't add null checks when the type system says a value cannot be null.

## Additional Resources

### Official Microsoft Documentation
- [Azure Functions Documentation](https://learn.microsoft.com/en-us/azure/azure-functions/)
- [Azure Functions Best Practices](https://learn.microsoft.com/en-us/azure/azure-functions/functions-best-practices)
- [Azure Functions Isolated Process Guide](https://learn.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide)
- [Working with Azure Functions in Visual Studio Code](https://learn.microsoft.com/en-us/azure/azure-functions/how-to-create-function-vs-code?pivots=programming-language-csharp)
- [Azure Functions Triggers and Bindings](https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings)
- [Azure Functions Performance and Reliability](https://learn.microsoft.com/en-us/azure/azure-functions/performance-reliability)

### Security and Authentication
- [Azure Functions security best practices](https://learn.microsoft.com/en-us/azure/azure-functions/functions-security-best-practices)
- [Secure Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-security-considerations)
- [Azure Functions authentication and authorization](https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-and-authorization)

### Development and Testing
- [Dependency Injection in Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-dotnet-dependency-injection)
- [Azure Functions local development](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-local)
- [Test Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-test-a-function)

### Deployment and Infrastructure
- [Deploy Azure Functions using GitHub Actions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-how-to-github-actions)
- [Azure Functions deployment](https://learn.microsoft.com/en-us/azure/azure-functions/functions-deployment-technologies)
- [Bicep Template for Function Apps](https://learn.microsoft.com/en-us/azure/templates/microsoft.web/sites)
- [Deploy Bicep with GitHub Actions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-github-actions)

### Monitoring and Observability
- [Azure Functions monitoring](https://learn.microsoft.com/en-us/azure/azure-functions/functions-monitoring)
- [Application Insights for Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-monitoring?tabs=cmd#application-insights-integration)
- [Azure Functions error handling and retries](https://learn.microsoft.com/en-us/azure/azure-functions/functions-error-handling-retries)

### Performance and Scaling
- [Manage Azure Functions Connections](https://learn.microsoft.com/en-us/azure/azure-functions/functions-manage-connections)
- [Azure Functions hosting options](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale)
- [Improve throughput performance of Python apps in Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/python-scale-performance-reference)

### Advanced Topics
- [Azure Functions with Virtual Networks](https://learn.microsoft.com/en-us/azure/azure-functions/functions-networking-options)
- [Durable Functions overview](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview)
- [Azure Functions Premium plan](https://learn.microsoft.com/en-us/azure/azure-functions/functions-premium-plan)