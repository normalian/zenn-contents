---
title: "Using the Work IQ API to Get Insights into Your Work"
emoji: "🦔"
type: "tech" # tech: technical article / idea: opinion
topics: ["MCP", "AI", "csharp"]
published: true
publication_name: "microsoft"
---

Have you been using Work IQ? Of the IQ offerings Microsoft is promoting—Fabric IQ, Foundry IQ, Work IQ, and Web IQ—Work IQ may well be the one we use most often while being the least aware that we are using it. Work IQ makes the data each person has accumulated in Microsoft 365 available as grounding context, much like RAG. That said, very few people have probably ever called it directly.

There is a good reason for that: when you use Microsoft 365 Copilot, it calls Work IQ behind the scenes. Work IQ retrieves context from your day-to-day work, reviews information such as Teams conversations and Outlook emails, and enables an agent to tell you things like, "Hey, you forgot to follow up on this item this week."

You can also call Work IQ explicitly from your own applications. As described in the following documentation, its APIs are available through A2A, MCP, and REST:
- [Work IQ overview](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/)

Microsoft has also published sample applications in C# and other languages. These samples show how you can call Work IQ directly from your own application:
- [GitHub - microsoft / work-iq-samples](https://github.com/microsoft/work-iq-samples/tree/main)

## Potential Real-World Use Cases

Work IQ's core capability—reading data stored in Microsoft 365—is extremely powerful. In practice, however, it means calling an API in the Entra ID tenant where your production Microsoft 365 environment is running.

In many projects, the Azure environment and Microsoft 365 environment use separate Entra ID tenants. In that case, the application needs to read information from an Entra ID tenant other than the one used by the Azure environment. This is entirely possible. You can create the service principal used for Entra ID access in the Microsoft 365 tenant, then configure the Azure environment to use its details. The architecture looks like this:

![](/images/workiq-sample-01/workiq-sample-architecture-01-en.png) 

As explained later, the Azure subscription used for Work IQ consumption-based billing must be linked to the Entra ID tenant for the Microsoft 365 environment. By using the service principal created in that environment from elsewhere, you can access Work IQ from web and client applications running virtually anywhere, whether in Azure, on-premises, AWS, or GCP.

## The Sample Application

In this example, we will call Work IQ directly from a C# console application. We will use its MCP server interface, although REST and A2A are also supported.

```txt

C# Console App
  │
  ├─ Microsoft Entra ID
  │    └─ Interactive user sign-in
  │         Scope: WorkIQAgent.Ask
  │
  └─ Work IQ MCP
       https://workiq.svc.cloud.microsoft/mcp

```

By accessing the `workiq.svc.cloud.microsoft` endpoint shown above, your application can take advantage of Work IQ.

Next, let's walk through the required configuration.

## Enable Work IQ API Billing

The first step is to enable billing at `admin.microsoft.com`. Open the Microsoft 365 admin center, go to **Home > Copilot > Cost management**, enable billing, and select the Azure subscription to be charged for Work IQ usage. You can only select an Azure subscription associated with the Entra ID tenant for your Microsoft 365 environment, so you may need to transfer the subscription to that tenant first.

- [Understand usage-based billing and cost management for Copilot Credits](https://learn.microsoft.com/en-us/microsoft-365/copilot/usage-based-billing-overview-copilot-credits)
- [Optimize Copilot Credit costs with a pre-purchase plan](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/copilot-credit-p3)

The following screenshot shows the portal after billing has been enabled successfully:

![](/images/workiq-sample-01/cost-billing-01.png) 

Once the configuration is complete, the **Configuration** tab displays the linked Azure subscription as shown below. Usage-based Work IQ charges will be billed to this subscription.

![](/images/workiq-sample-01/cost-billing-02.png) 

That completes the configuration in the Microsoft 365 admin center.

## Create a Service Principal in the Microsoft 365 Entra ID Tenant

Before creating the service principal for the C# application, you need to enable Work IQ itself in addition to configuring billing. Follow the instructions in [Microsoft Learn - Enable your tenant for Work IQ](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/enable-work-iq?tabs=entra-admin), sign in to the Entra ID tenant where you plan to use Work IQ, and run the following command:

```bash

az ad sp create --id fdcc1f02-fc51-4226-8753-f668596af7f7

```

This command creates the service principal for Work IQ itself, making Work IQ available to your application. If you skip this step, you will not be able to assign Work IQ permissions to your application's service principal.


Next, create the service principal that the C# application will use. Configure it as follows:

- Create a single-tenant app registration
- Configure it as a mobile and desktop application
- Set the redirect URI to `http://localhost`
- Add the delegated permission for the Work IQ API
- Select `WorkIQAgent.Ask`
- Grant admin consent

Adding the delegated permission for the Work IQ API is especially important. With delegated authentication, the application displays a sign-in prompt and operates using the signed-in user's identity.

## Run the C# Application

Now we can run the application using the service principal configured above. In this example, the prompt asks Work IQ to review information from the past seven days—including Teams messages and email—and identify tasks that may still be unfinished. At the time of writing, there does not appear to be an SDK available, so the application calls the endpoint directly.

```csharp

using Microsoft.Identity.Client;
using System.Net.Http.Headers;
using System.Text;

const string tenantId =
    "your-tenant-id";

const string clientId =
    "your-client-id";

string[] scopes =
{
    "api://workiq.svc.cloud.microsoft/WorkIQAgent.Ask"
};

Console.WriteLine("Signing in...");

var app =
    PublicClientApplicationBuilder
        .Create(clientId)
        .WithTenantId(tenantId)
        .WithRedirectUri("http://localhost")
        .Build();

var auth =
    await app
        .AcquireTokenInteractive(scopes)
        .ExecuteAsync();

Console.WriteLine($"User    : {auth.Account?.Username}");
Console.WriteLine($"Tenant  : {auth.TenantId}");
Console.WriteLine($"Expires : {auth.ExpiresOn}");
Console.WriteLine();

var httpClient = new HttpClient();

httpClient.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue(
        "Bearer",
        auth.AccessToken);

//
httpClient.DefaultRequestHeaders.Accept.Clear();

httpClient.DefaultRequestHeaders.Accept.Add(
    new MediaTypeWithQualityHeaderValue(
        "application/json"));

httpClient.DefaultRequestHeaders.Accept.Add(
    new MediaTypeWithQualityHeaderValue(
        "text/event-stream"));


//
// 1. tools/list
//
var toolsListRequest =
"""
{
  "jsonrpc":"2.0",
  "id":1,
  "method":"tools/list"
}
""";

Console.WriteLine();
Console.WriteLine("Calling tools/list...");

var toolsResponse =
    await httpClient.PostAsync(
        "https://workiq.svc.cloud.microsoft/mcp",
        new StringContent(
            toolsListRequest,
            Encoding.UTF8,
            "application/json"));

var toolsBody =
    await toolsResponse.Content.ReadAsStringAsync();

Console.WriteLine(toolsBody);

//
// 2. ask
//
var askRequest =
"""
{
  "jsonrpc":"2.0",
  "id":2,
  "method":"tools/call",
  "params":
  {
    "name":"ask",
    "arguments":
    {
      "question":"Review my last 7 days of work and identify forgotten or unfinished tasks. Include emails, meetings, Teams messages, documents, and Planner tasks. Return the top 10 items ordered by urgency."
    }
  }
}
""";

Console.WriteLine();
Console.WriteLine("Calling ask...");

var askResponse =
    await httpClient.PostAsync(
        "https://workiq.svc.cloud.microsoft/mcp",
        new StringContent(
            askRequest,
            Encoding.UTF8,
            "application/json"));

var askBody =
    await askResponse.Content.ReadAsStringAsync();

Console.WriteLine();
Console.WriteLine("===== ANSWER =====");
Console.WriteLine(askBody);


//
// 3. fetch
//
var fetchRequest = """
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params":
  {
    "name": "fetch",
    "arguments":
    {
      "entityUrls":
      [
        "/me/messages?$select=subject,receivedDateTime,from&$top=5"
      ]
    }
  }
}
""";

Console.WriteLine();
Console.WriteLine("Calling fetch...");

var fetchResponse = await httpClient.PostAsync(
    "https://workiq.svc.cloud.microsoft/mcp",
    new StringContent(fetchRequest, Encoding.UTF8, "application/json"));

Console.WriteLine($"HTTP {(int)fetchResponse.StatusCode}");

var fetchBody = await fetchResponse.Content.ReadAsStringAsync();

Console.WriteLine();
Console.WriteLine("==== FETCH RESULT ====");
Console.WriteLine(fetchBody);

Console.WriteLine();
Console.WriteLine("Done.");

```

The output looks like this:


```txt

Calling tools/list...
event: message
data: {"result":{"tools":[{"name":"fetch_blob","description":"Fetch a binary file (document, image, photo) from a WorkIQ path \u2014 PDFs,Office files, profile photos, etc. Up to 4 MB for the raw file. Returns the file\u0027s bytes base64-encoded plus metadata.","inputSchema":{"type":"object","properties":{"path":{"description":"Relative WorkIQ path to the binary content, e.g. /me/photo/$value or /drives/{id}/items/{id}/content. Must be a relative path \u2014 do not include a base URL.","type":"string"},"format":{"description":"Optional. Forwarded as $format to convert the item on download (e.g. \u0022pdf\u0022); honored only on drive-content endpoints that accept it.","type":["string","null"],"default":null},"agentId":{"description":"Optional agent ID to target a specific agent.","type":["string","null"],"default":null}},"requ<<output omitted>>


Calling ask...

===== ANSWER =====
event: message
data: {"result":{"content":[{"type":"text","text":"I could not reliably retrieve recent meetings, Planner tasks, documents, or Teams activity from the last 7 days. The available grounding data primarily contains email activity, so the list below is limited to items that explicitly appear as open actions, unread messages, recommendations, or follow-up candidates in your mailbox. I am not inferring tasks that are not explicitly supported by the retrieved data.\n\n### Top 10 Potentially Forgotten or Unfinished Items (ordered by urgency)\n\n1. **Review and apply the Azure Logic App recommendation fo
<<output omitted>>

Calling fetch...
HTTP 200

==== FETCH RESULT ====
event: message
data: {"result":{"content":[],"structuredContent":{"results":[{"data":{"@odata.context":
<<output omitted>>

Done.

```

### Troubleshooting an Error

I encountered an issue where all of the following worked successfully:

- Entra ID authentication
- Acquiring the `WorkIQAgent.Ask` token
- MCP initialization
- `tools/list`

However, the `ask` and `fetch` operations both returned a billing error:

```txt

caller tenant billing policy not configured for WorkIQ

```

After investigating, I found that the issue was not related to application authentication or API permissions. The usage-based billing policy had not been configured for the Microsoft 365 tenant. Enabling usage-based billing under **Microsoft 365 admin center > Copilot > Cost management** resolved the error.
