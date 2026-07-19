---
title: "Building an Copilot Agent with Agent365 CLI"
emoji: "🐈"
type: "tech" # tech: technical article / idea: idea
topics: ["Agent365", "AI"]
published: true
publication_name: "microsoft"
---

# Introduction

In this article, I walk through how to use Agent 365 CLI (a365 CLI), provided by Microsoft, together with Microsoft Agent Framework to deploy and publish an enterprise AI agent on Azure that can integrate with Microsoft 365 Teams.

Agent development with generative AI is evolving rapidly. When the requirement is to securely integrate with Microsoft 365 under organizational identity and permission governance, you need a governance-aware architecture that goes beyond a simple web API chatbot.

Microsoft Agent 365 (A365) acts as a control plane that manages these AI agents as enterprise identities based on Microsoft Entra ID (formerly Azure AD), and securely integrates them with Microsoft 365 workloads such as Teams, Outlook, and Copilot. A key benefit of Agent 365 CLI is that it makes it easy to publish agents in a way that can be managed and operated through Agent 365.

Agent 365 CLI (a365 CLI) is a command-line tool that automates the development lifecycle for Agent 365-compatible agents, enabling you to run the following end-to-end:

- Create an agent Blueprint (template)
- Create an Agent Identity in Microsoft Entra ID
- Automatically provision infrastructure such as Azure App Service
- Deploy agent code to Azure
- Generate a publish package for Microsoft 365 admin center
- Create an Agent Instance and assign licenses

This article uses the following official sample application as the base and explains the process through publication.

Microsoft Agent Framework Sample Agent (.NET)
https://github.com/microsoft/Agent365-Samples/tree/main/dotnet/agent-framework/sample-agent

This sample is a minimal enterprise-ready AI agent that:

- Uses Microsoft Agent Framework as the orchestrator
- Combines Microsoft Agent 365 SDK and Microsoft 365 Agents SDK
- Hosts the agent on ASP.NET Core
- Receives messages from Microsoft 365 through the `/api/messages` endpoint

---

## Core Resource Model in Agent 365

In Agent 365, unlike a typical bot application, agents are managed as dedicated resources in Microsoft Entra ID. It can be a bit confusing at first, but the resources created primarily by Agent365 CLI are organized in the following three layers.

### 1. Agent Blueprint

An Agent Blueprint is a template that defines the agent identity, permissions, and infrastructure configuration, including:

- Application registration for the agent
- Access permissions to Microsoft Graph API and others
- Permissions for Messaging Bot API
- Access permissions to MCP (Model Context Protocol) servers
- Policy settings inherited by Agent Instances

All Agent Instances are created based on a Blueprint.

---

### 2. Agent Identity

An Agent Identity is a service principal created in Microsoft Entra ID. Similar to a human user, it has:

- A unique Object ID
- App ID
- API access permissions
- Sponsor (owner)
- Manager information

It operates as the security principal that accesses Microsoft 365 workloads.

Unlike a simple backend app for Azure Bot Service, Agent Identity is treated as an enterprise identity subject to organizational policy, audit, and Conditional Access.

### 3. Agent Instance

An Agent Instance is the runtime entity created from a Blueprint. By running the `create-instance` command in Agent365 CLI, the following are configured automatically:

- Agent User (Entra ID user) associated with the Agent Identity
- Microsoft 365 license assignment
- Agent sponsor settings

This makes the digital worker available for @mentions in Microsoft Teams and Copilot.

### Azure Resources Created

When you run the Agent365 CLI `setup` command, in addition to Microsoft Entra ID resources, Azure-side runtime resources are provisioned automatically. The main resources are:

- App Service Plan
- Azure Web App
- Managed Identity (for Web App)
- App registration associated with the Agent Blueprint
- Microsoft Graph API permission settings
- Messaging Bot API permission settings

The agent deployed to Azure Web App receives activities from Microsoft 365 (Teams / Copilot) via the `/api/messages` endpoint and processes them through Agent Framework.

---

In this article, I will explain step by step how these resources are built with a365 CLI and integrated with Microsoft 365 using the sample application.

## Prerequisites

Before following the steps in this article, complete the following:

- Create a Service Principal in Microsoft Entra ID
- Acquire an Azure subscription and create a Resource Group
- Install Agent365 CLI
- Get the official sample application used in this article: https://github.com/microsoft/Agent365-Samples.git

Use the following commands to install Agent365 CLI. After installation, verify with `-h`.

```bash
dotnet tool install --global Microsoft.Agents.A365.DevTools.Cli --prerelease
a365 -h
```

# Create and Publish the Agent

## Configure Service Principal and Create Agent Config File

Using Agent 365 CLI, the process to publish an agent to Microsoft 365 can be broken down into three major steps: Agent365 CLI configuration, environment setup for Agent Blueprint and Azure resources, and publish package generation.

First, initialize Agent365 CLI settings with the following command.

```bash
a365 config init
```

You will be prompted to confirm Service Principal permissions as shown below.
![](/images/a365cli-agentdevelopment-01/sp-setting-01.png)

If permissions are already configured, you can proceed. Otherwise, go to Entra ID and grant the required permissions. Note that Microsoft Graph must be granted as Delegated Permissions, not Application Permissions.
![](/images/a365cli-agentdevelopment-01/sp-setting-02.png)

After granting permissions, click Grant admin consent to delegate them. Then run `a365 config init` again and enter the target Service Principal client ID. Agent365 CLI detects your Azure subscription, Resource Group, App Service Plan, and deployment region, then generates the configuration file (`a365.config.json`).
![](/images/a365cli-agentdevelopment-01/a365-init-01.png)

At the same time, it validates that the custom client app registration exists in Microsoft Entra ID and that required permissions plus admin consent are in place.

## Provision Infrastructure Resources

Next, run the following command to provision the Agent365 runtime environment in Azure and Microsoft Entra ID.

```bash
a365 setup all
```

This command runs Agent365 initial setup in one shot. On Azure, it creates infrastructure such as App Service, App Service Plan, and Managed Identity. On Microsoft Entra ID, it registers Agent Blueprint and configures inherited permissions for Graph / Messaging Bot API and more. It also generates `a365.generated.config.json`, which stores values such as Blueprint ID and Messaging Endpoint.

![](/images/a365cli-agentdevelopment-01/a365-setupall-01.png)

## Create Blueprint via Package Registration in Microsoft Admin Center

After setup completes, generate the publish package for upload to Microsoft 365 admin center with:

```bash
a365 publish
```

![](/images/a365cli-agentdevelopment-01/a365-publish-01.png)

Running this updates `manifest.json` with the Agent Blueprint ID and generates `manifest.zip` along with icon files.

Upload this `manifest.zip` from Microsoft 365 admin center (Agents > All agents > Upload custom agent) at https://admin.microsoft.com/ to publish it within your organization as an agent available in Microsoft Teams and Copilot. You can also set the target users for publication as shown below.
![](/images/a365cli-agentdevelopment-01/a365-admincenter-01.png)
![](/images/a365cli-agentdevelopment-01/a365-admincenter-02.png)

After `manifest.zip` upload completes, the Agent Blueprint is registered as shown below.

## Register the Endpoint in Teams Developer Portal

Next, register the application endpoint that receives agent messages in Teams Developer Portal. First, run this command to check the endpoint:

```bash
a365 config display -g
```

![](/images/a365cli-agentdevelopment-01/teams-devcenter-01.png)

Save the displayed `/api/messages` URL, then open Teams Developer Portal: https://dev.teams.microsoft.com/home. In Tools, select your Agent Blueprint, then in Configuration set Agent Type to API Based, enter the saved URL in Notification URL, and save.

![](/images/a365cli-agentdevelopment-01/teams-devcenter-02.png)

## Create Agent Instance and Identity

Now create the actual runtime entity for the agent. A Blueprint is the design template, so this is where the actual instance gets created.

```bash
a365 create-instance
```

![](/images/a365cli-agentdevelopment-01/a365-createinstace-01.png)

When this completes, you will see values such as instance ID and agent ID, as shown below. In addition, `appsettings.json` is updated with generated values such as client secret.

## Deploy ASP.NET Core Application to App Service

Before deployment, update `appsettings.json`. The following sample shows the fields to update. In addition to Entra ID-related values such as Service Principal client ID, Agent Blueprint ID, and tenant ID, Azure OpenAI settings are also required.

```json
{
  "AgentApplication": {
    "StartTypingTimer": false,
    "RemoveRecipientMention": false,
    "NormalizeMentions": false,
    "AgenticAuthHandlerName": "agentic",
    "UserAuthorization": {
      "AutoSignin": false,
      "Handlers": {
        "agentic": {
          "Type": "AgenticUserAuthorization",
          "Settings": {
            "Scopes": [
              "https://graph.microsoft.com/.default"
            ],
            "AlternateBlueprintConnectionName": "ServiceConnection"
          }
        }
      }
    }
  },
  "TokenValidation": {
    "Audiences": [
      "<your-service-principal-client-id>"
    ],
    "Enabled": false,
    "TenantId": "<your-entraid-tenant-id>"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.Agents": "Warning",
      "Microsoft.Hosting.Lifetime": "Information",
      "Microsoft.SemanticKernel*": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Connections": {
    "ServiceConnection": {
      "Settings": {
        "AuthType": "ClientSecret", // This AuthType is required to update from UserManagedIdentity to ClientSecret
        "AuthorityEndpoint": "https://login.microsoftonline.com/<your-entraid-tenant-id>",
        "ClientId": "<your-service-principal-client-id>",
        "Scopes": [
          "5a807f24-c9de-44ee-a3a7-329e88a00ffc/.default"
        ],
        "ClientSecret": "<your-clientid-but-this-id-should-be-generated>",
        "AgentId": "<your-agentid-but-this-id-should-be-generated>"
      }
    }
  },
  "ConnectionsMap": [
    {
      "ServiceUrl": "*",
      "Connection": "ServiceConnection"
    }
  ],
  "AIServices": {
    "AzureOpenAI": {
      "DeploymentName": "gpt-4.1-mini",
      "Endpoint": "<https://your-azure-openai-url.openai.azure.com/>",
      "ApiKey": "<your-azure-openai-api-key>"
    }
  },
  "OpenWeatherApiKey": "<your-api-key>"
}
```

After updating the configuration file, deploy the application with:

```bash
a365 deploy
```

If the command runs successfully, you will see output like below.

![](/images/a365cli-agentdevelopment-01/a365-deploy-01.png)

# Validation

If all settings are configured correctly, an agent named "(your agent name) Agent User" is created on the Teams side. Find that agent in Teams chat and send a message; you should receive a response like the following.

![](/images/a365cli-agentdevelopment-01/a365-chatwithagent-01.png)

# References

- Agent 365 CLI documentation: https://learn.microsoft.com/en-us/microsoft-agent-365/developer/agent-365-cli
- Agent365 Samples repository: https://github.com/microsoft/Agent365-Samples
- Teams Developer Portal: https://dev.teams.microsoft.com/home
- Microsoft 365 admin center: https://admin.microsoft.com/
- OpenWeather pricing: https://openweathermap.org/price
