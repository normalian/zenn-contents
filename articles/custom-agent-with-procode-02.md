---
title: "Azure Bot Service を Teams Channel で公開する際、エンドポイント保護はどうしたらよいか？"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Teams", "AI", "SemanticKernel"]
published: true
publication_name: "microsoft"
---

## はじめに

今回は以下の記事を実施済なのが前提となっているので、どちらかの記事は必ず通読しておいてください。
- [Develop Custom Engine Agent to Microsoft 365 Copilot Chat with pro-code](https://techcommunity.microsoft.com/blog/appsonazureblog/develop-custom-engine-agent-to-microsoft-365-copilot-chat-with-pro-code/4435612)
- [カスタム エンジン エージェントを Microsoft 365 Copilot で公開する](https://ayuina.github.io/ainaba-csa-blog/fy26/mcs-custom-engine-agent/)

上記の記事では C# 等の Pro code を使って Custom Engine Agent を公開できまることを解説しましたが、今回はセキュリティの観点から各エンドポイントをどの様に保護したら良いかを考えてみたいと思います。色んな思考実験をした結果で「これが良さそうだ」というところまではたどり着きましたが、これでセキュリティが担保可能かどうかはご自身の環境を含めて精査をお願い致します。

## どこのエンドポイントを制御できるのか？

今回の構成で気になるエンドポイントは以下の三つという認識です。
- Teams のエンドポイント
- Azure Bot Service のエンドポイント
- ASP.NET Core のエンドポイント（元記事では devtunnel ローカルですが、現実では App Service 辺りでしょうか？
![](/images/custom-agent-with-procode-02/customagent-architecture-01.png) 

Teams のエンドポイント制御 に関しては Teams 自体のアプリ管理に帰結する認識です。Teams のマニュフェストファイルを自社の M365 テナントに登録する必要があり、Teams Admin ポータルの制御となります。エンドポイントの制御というより、誰が Custom Agent（という名の Teams アプリ）にアクセスできるのかを制限するので、ここでは悪意のあるユーザ側への制御はユーザ単位では可能だが、エンドポイント制御はできず悪意のある組織がアプリモジュールを盗み出した場合の制御に気になる点があることが分ります。
Azure Bot Service のエンドポイントに関しては Teams Channel と ASP.NET Core 側のエンドポイントをつなぐだけであり、ここでの制御は Service Principal を指定するのみです。
一方で ASP.NET Core のエンドポイント側について考えてみましょう。Azure Bot Service 上の設定で ASP.NET Core のエンドポイントを公開しますが、これはインターネット上でパブリックなエンドポイントです。以下の様に network isolation 向けの記事も存在しますが、Azure Bot Service が DirectLine の場合にのみ利用可能であり、Teams Channel の場合は利用できません。
https://learn.microsoft.com/en-us/azure/bot-service/dl-network-isolation-concept?view=azure-bot-service-4.0

その他の記事も確認してみましょう。以下の記事は両方とも同じことを解説してくれていますが、ポイントとしては「Teams 自体が SaaS アプリケーションなので、Teams 経由でアクセスする Bot のエンドポイント（例：https://my-webapp-endpoint.net/api/messages）は公開されている必要がある」と述べています。
- https://learn.microsoft.com/en-us/answers/questions/2263616/is-it-possible-to-integrate-azure-bot-with-teams-w
- https://learn.microsoft.com/en-us/answers/questions/2153606/how-to-create-azure-bot-service-in-a-private-netwo

つまるところ、ネットワークレベルでの完全分離は厳しいということになります。こうなると ASP.NET Core 側で bearer header token を見ての制御が必要となりそうなので、C# コードを見ていきたいと思います。

## ASP.NET Core アプリ側でのエンドポイント制御

ではアプリケーションレベルでの制御はどうなっているのでしょうか？ ASP.NET Core のアプリ側にもどってテンプレートプロジェクトを眺めてみたいと思いますが、特に Program.cs に以下の記載があったことを覚えていますでしょうか？

```csharp

// 中略

// Register the WeatherForecastAgent
builder.Services.AddTransient<WeatherForecastAgent>();

// Add AspNet token validation - ★★ここ★★
builder.Services.AddBotAspNetAuthentication(builder.Configuration);

// Register IStorage.  For development, MemoryStorage is suitable.
// For production Agents, persisted storage should be used so
// that state survives Agent restarts, and operate correctly
// in a cluster of Agent instances.
builder.Services.AddSingleton<IStorage, MemoryStorage>();

// 中略

```

上記の AddBotAspNetAuthentication は、実は同プロジェクト内にある AspNetExtensions.cs に実態が存在し、こちらで Access Token の制御をしています。今回はこちらの中身を見てみましょう。 AspNetExtensions.cs ファイルの AddBotAspNetAuthentication メソッドの抜粋は以下となります。

```csharp

    public static void AddBotAspNetAuthentication(this IServiceCollection services, IConfiguration configuration, string tokenValidationSectionName = "TokenValidation", ILogger logger = null)
    {
        IConfigurationSection tokenValidationSection = configuration.GetSection(tokenValidationSectionName);
        List<string> validTokenIssuers = tokenValidationSection.GetSection("ValidIssuers").Get<List<string>>();
        List<string> audiences = tokenValidationSection.GetSection("Audiences").Get<List<string>>();

        if (!tokenValidationSection.Exists())
        {
            logger?.LogError("Missing configuration section '{tokenValidationSectionName}'. This section is required to be present in appsettings.json",tokenValidationSectionName);
            throw new InvalidOperationException($"Missing configuration section '{tokenValidationSectionName}'. This section is required to be present in appsettings.json");
        }

        // If ValidIssuers is empty, default for ABS Public Cloud
        if (validTokenIssuers == null || validTokenIssuers.Count == 0)
        {
            validTokenIssuers =
            [
                "https://api.botframework.com",
                "https://sts.windows.net/d6d49420-f39b-4df7-a1dc-d59a935871db/",
                "https://login.microsoftonline.com/d6d49420-f39b-4df7-a1dc-d59a935871db/v2.0",
                "https://sts.windows.net/f8cdef31-a31e-4b4a-93e4-5f571e91255a/",
                "https://login.microsoftonline.com/f8cdef31-a31e-4b4a-93e4-5f571e91255a/v2.0",
                "https://sts.windows.net/69e9b82d-4842-4902-8d1e-abc5b98a55e8/",
                "https://login.microsoftonline.com/69e9b82d-4842-4902-8d1e-abc5b98a55e8/v2.0",
            ];

            string tenantId = tokenValidationSection["TenantId"];
            if (!string.IsNullOrEmpty(tenantId))
            {
                validTokenIssuers.Add(string.Format(CultureInfo.InvariantCulture, AuthenticationConstants.ValidTokenIssuerUrlTemplateV1, tenantId));
                validTokenIssuers.Add(string.Format(CultureInfo.InvariantCulture, AuthenticationConstants.ValidTokenIssuerUrlTemplateV2, tenantId));
            }
        }

        if (audiences == null || audiences.Count == 0)
        {
            throw new ArgumentException($"{tokenValidationSectionName}:Audiences requires at least one value");
        }

        bool isGov = tokenValidationSection.GetValue("IsGov", false);
        bool azureBotServiceTokenHandling = tokenValidationSection.GetValue("AzureBotServiceTokenHandling", true);

        // If the `AzureBotServiceOpenIdMetadataUrl` setting is not specified, use the default based on `IsGov`.  This is what is used to authenticate ABS tokens.
        string azureBotServiceOpenIdMetadataUrl = tokenValidationSection["AzureBotServiceOpenIdMetadataUrl"];
        if (string.IsNullOrEmpty(azureBotServiceOpenIdMetadataUrl))
        {
            azureBotServiceOpenIdMetadataUrl = isGov ? AuthenticationConstants.GovAzureBotServiceOpenIdMetadataUrl : AuthenticationConstants.PublicAzureBotServiceOpenIdMetadataUrl;
        }

        // If the `OpenIdMetadataUrl` setting is not specified, use the default based on `IsGov`.  This is what is used to authenticate Entra ID tokens.
        string openIdMetadataUrl = tokenValidationSection["OpenIdMetadataUrl"];
        if (string.IsNullOrEmpty(openIdMetadataUrl))
        {
            openIdMetadataUrl = isGov ? AuthenticationConstants.GovOpenIdMetadataUrl : AuthenticationConstants.PublicOpenIdMetadataUrl;
        }

        TimeSpan openIdRefreshInterval = tokenValidationSection.GetValue("OpenIdMetadataRefresh", BaseConfigurationManager.DefaultAutomaticRefreshInterval);

        _ = services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
            options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
        })
        .AddJwtBearer(options =>
        {
            options.SaveToken = true;
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ClockSkew = TimeSpan.FromMinutes(5),
                ValidIssuers = validTokenIssuers,
                ValidAudiences = audiences,
                ValidateIssuerSigningKey = true,
                RequireSignedTokens = true,
            };

            // Using Microsoft.IdentityModel.Validators
            options.TokenValidationParameters.EnableAadSigningKeyIssuerValidation();

            options.Events = new JwtBearerEvents
            {
                // Create a ConfigurationManager based on the requestor.  This is to handle ABS non-Entra tokens.
                OnMessageReceived = async context =>
                {
                    string authorizationHeader = context.Request.Headers.Authorization.ToString();

                    if (string.IsNullOrEmpty(authorizationHeader))
                    {
                        // Default to AadTokenValidation handling
                        context.Options.TokenValidationParameters.ConfigurationManager ??= options.ConfigurationManager as BaseConfigurationManager;
                        await Task.CompletedTask.ConfigureAwait(false);
                        return;
                    }

                    string[] parts = authorizationHeader?.Split(' ');
                    if (parts.Length != 2 || parts[0] != "Bearer")
                    {
                        // Default to AadTokenValidation handling
                        context.Options.TokenValidationParameters.ConfigurationManager ??= options.ConfigurationManager as BaseConfigurationManager;
                        await Task.CompletedTask.ConfigureAwait(false);
                        return;
                    }

                    JwtSecurityToken token = new(parts[1]);
                    string issuer = token.Claims.FirstOrDefault(claim => claim.Type == AuthenticationConstants.IssuerClaim)?.Value;

                    if (azureBotServiceTokenHandling && AuthenticationConstants.BotFrameworkTokenIssuer.Equals(issuer))
                    {
                        // Use the Bot Framework authority for this configuration manager
                        context.Options.TokenValidationParameters.ConfigurationManager = _openIdMetadataCache.GetOrAdd(azureBotServiceOpenIdMetadataUrl, key =>
                        {
                            return new ConfigurationManager<OpenIdConnectConfiguration>(azureBotServiceOpenIdMetadataUrl, new OpenIdConnectConfigurationRetriever(), new HttpClient())
                            {
                                AutomaticRefreshInterval = openIdRefreshInterval
                            };
                        });
                    }
                    else
                    {
                        context.Options.TokenValidationParameters.ConfigurationManager = _openIdMetadataCache.GetOrAdd(openIdMetadataUrl, key =>
                        {
                            return new ConfigurationManager<OpenIdConnectConfiguration>(openIdMetadataUrl, new OpenIdConnectConfigurationRetriever(), new HttpClient())
                            {
                                AutomaticRefreshInterval = openIdRefreshInterval
                            };
                        });
                    }

                    await Task.CompletedTask.ConfigureAwait(false);
                },
                OnTokenValidated = context =>
                {
                    logger?.LogDebug("TOKEN Validated");
                    return Task.CompletedTask;
                },
                OnForbidden = context =>
                {
                    logger?.LogWarning("Forbidden: {m}", context.Result.ToString());
                    return Task.CompletedTask;
                },
                OnAuthenticationFailed = context =>
                {
                    logger?.LogWarning("Auth Failed {m}", context.Exception.ToString());
                    return Task.CompletedTask;
                }
            };
        });
    }

```

上記のコードは appsettings.your-env.json のファイルから構成情報を読み取り、トークンバリデーションで利用していることが分かります。特に TokenValidationParameters の ValidAudiences = audiences を設定していることで、Azure Bot Service で利用している Service Principal 以外のアクセスはバリデーションで弾かれていることが分かります。

一方で、以下のコードで気づいたのはトークンが発行されない場合はスルーパスしているという点です。つまり Service Principal 側の認可設定が不十分でトークンが飛ばない場合も処理が進んでしまうということが分かります。

```csharp

                OnMessageReceived = async context =>
                {
                    string authorizationHeader = context.Request.Headers.Authorization.ToString();

                    if (string.IsNullOrEmpty(authorizationHeader))
                    {
                        // Default to AadTokenValidation handling
                        context.Options.TokenValidationParameters.ConfigurationManager ??= options.ConfigurationManager as BaseConfigurationManager;
                        await Task.CompletedTask.ConfigureAwait(false);
                        return;
                    }

```

加えて、こちらのコードで気になる点としては「Custom Engine Agent のアプリをコピーして持ち出し、別の Entra ID テナントでアプリをアップロードした場合でも利用できる」という点です（まぁ、これは複数社に Custom Engine Agent のサービスを提供することを前提とした構成だからでしょうが）。

したがって、以下の二点を直したくなります。
- トークンが空の場合はエラーにしたい
- 知らない Entra ID テナントからのアクセスは弾きたい

上記の二つを行う場合、そもそも Service Principal 設定をかえなければいけません。Service Principal の API permissions のタブを開き、以下の様に User.Read.All の権限を付与してください。実はこれがないと Access Token が飛ばないので、トークン自体の検証ができなくなります。

次に Service Principal の認可設定後に ASP.NET Core アプリを動かし、以下のコード辺りで bread point を設定して authorizationHeader に含まれるトークンの中身を見てみます。

```csharp

string authorizationHeader = context.Request.Headers.Authorization.ToString();

```

BASE64 でエンコードされていますが、ChatGPT さんにでコードしてもらって中身を確認しましょう。

![](/images/custom-agent-with-procode-02/token-base640-decode.png) 

いくつかぬりつぶさせてもらっていますが、aud に Service Principal の client iD が含まれており、serviceurl に Entra ID の tenant id が含まれていることが分かります（Tenant ID 自体を Access Token に含める様に認可設定をしようとしましたが、今回はちょっと力及ばず）。

以下に「トークンが空の場合はエラーにしたい」と「知らない Entra ID テナントからのアクセスは弾きたい」を両方とも実装したサンプルを提示します。どこを直したかはコメントを見てもらえば一目瞭然かと。

```csharp

    public static void AddBotAspNetAuthentication(this IServiceCollection services, IConfiguration configuration, string tokenValidationSectionName = "TokenValidation", ILogger logger = null)
    {
        IConfigurationSection tokenValidationSection = configuration.GetSection(tokenValidationSectionName);
        List<string> validTokenIssuers = tokenValidationSection.GetSection("ValidIssuers").Get<List<string>>();
        List<string> audiences = tokenValidationSection.GetSection("Audiences").Get<List<string>>();

        if (!tokenValidationSection.Exists())
        {
            logger?.LogError("Missing configuration section '{tokenValidationSectionName}'. This section is required to be present in appsettings.json",tokenValidationSectionName);
            throw new InvalidOperationException($"Missing configuration section '{tokenValidationSectionName}'. This section is required to be present in appsettings.json");
        }

        // If ValidIssuers is empty, default for ABS Public Cloud
        if (validTokenIssuers == null || validTokenIssuers.Count == 0)
        {
            validTokenIssuers =
            [
                "https://api.botframework.com",
                "https://sts.windows.net/d6d49420-f39b-4df7-a1dc-d59a935871db/",
                "https://login.microsoftonline.com/d6d49420-f39b-4df7-a1dc-d59a935871db/v2.0",
                "https://sts.windows.net/f8cdef31-a31e-4b4a-93e4-5f571e91255a/",
                "https://login.microsoftonline.com/f8cdef31-a31e-4b4a-93e4-5f571e91255a/v2.0",
                "https://sts.windows.net/69e9b82d-4842-4902-8d1e-abc5b98a55e8/",
                "https://login.microsoftonline.com/69e9b82d-4842-4902-8d1e-abc5b98a55e8/v2.0",
            ];

            string tenantId = tokenValidationSection["TenantId"];
            if (!string.IsNullOrEmpty(tenantId))
            {
                validTokenIssuers.Add(string.Format(CultureInfo.InvariantCulture, AuthenticationConstants.ValidTokenIssuerUrlTemplateV1, tenantId));
                validTokenIssuers.Add(string.Format(CultureInfo.InvariantCulture, AuthenticationConstants.ValidTokenIssuerUrlTemplateV2, tenantId));
            }
        }

        if (audiences == null || audiences.Count == 0)
        {
            throw new ArgumentException($"{tokenValidationSectionName}:Audiences requires at least one value");
        }

        bool isGov = tokenValidationSection.GetValue("IsGov", false);
        bool azureBotServiceTokenHandling = tokenValidationSection.GetValue("AzureBotServiceTokenHandling", true);

        // If the `AzureBotServiceOpenIdMetadataUrl` setting is not specified, use the default based on `IsGov`.  This is what is used to authenticate ABS tokens.
        string azureBotServiceOpenIdMetadataUrl = tokenValidationSection["AzureBotServiceOpenIdMetadataUrl"];
        if (string.IsNullOrEmpty(azureBotServiceOpenIdMetadataUrl))
        {
            azureBotServiceOpenIdMetadataUrl = isGov ? AuthenticationConstants.GovAzureBotServiceOpenIdMetadataUrl : AuthenticationConstants.PublicAzureBotServiceOpenIdMetadataUrl;
        }

        // If the `OpenIdMetadataUrl` setting is not specified, use the default based on `IsGov`.  This is what is used to authenticate Entra ID tokens.
        string openIdMetadataUrl = tokenValidationSection["OpenIdMetadataUrl"];
        if (string.IsNullOrEmpty(openIdMetadataUrl))
        {
            openIdMetadataUrl = isGov ? AuthenticationConstants.GovOpenIdMetadataUrl : AuthenticationConstants.PublicOpenIdMetadataUrl;
        }

        TimeSpan openIdRefreshInterval = tokenValidationSection.GetValue("OpenIdMetadataRefresh", BaseConfigurationManager.DefaultAutomaticRefreshInterval);

        _ = services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
            options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
        })
        .AddJwtBearer(options =>
        {
            options.SaveToken = true;
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,                // this option enables to validate the audience claim with audiences values
                ValidateLifetime = true,
                ClockSkew = TimeSpan.FromMinutes(5),
                ValidIssuers = validTokenIssuers,
                ValidAudiences = audiences,
                ValidateIssuerSigningKey = true,
                RequireSignedTokens = true,
            };

            // Using Microsoft.IdentityModel.Validators
            options.TokenValidationParameters.EnableAadSigningKeyIssuerValidation();

            string tenantId = tokenValidationSection["TenantId"]; // テナント ID バリデーション用に変数格納
            options.Events = new JwtBearerEvents
            {
                // Create a ConfigurationManager based on the requestor.  This is to handle ABS non-Entra tokens.
                OnMessageReceived = async context =>
                {
                    string authorizationHeader = context.Request.Headers.Authorization.ToString();

                    if (string.IsNullOrEmpty(authorizationHeader))
                    {
                        // Default to AadTokenValidation handling
                        // context.Options.TokenValidationParameters.ConfigurationManager ??= options.ConfigurationManager as BaseConfigurationManager;
                        // await Task.CompletedTask.ConfigureAwait(false);
                        // return;
                        //
                        // トークンが空の場合はエラーにしたい
                        context.Fail("Authorization header is missing.");
                        logger?.LogWarning("Authorization header is missing.");
                        return;
                    }

                    string[] parts = authorizationHeader?.Split(' ');
                    if (parts.Length != 2 || parts[0] != "Bearer")
                    {
                        // Default to AadTokenValidation handling
                        context.Options.TokenValidationParameters.ConfigurationManager ??= options.ConfigurationManager as BaseConfigurationManager;
                        await Task.CompletedTask.ConfigureAwait(false);
                        return;
                    }

                    JwtSecurityToken token = new(parts[1]);
                    string issuer = token.Claims.FirstOrDefault(claim => claim.Type == AuthenticationConstants.IssuerClaim)?.Value;

                    if (azureBotServiceTokenHandling && AuthenticationConstants.BotFrameworkTokenIssuer.Equals(issuer))
                    {
                        // Use the Bot Framework authority for this configuration manager
                        context.Options.TokenValidationParameters.ConfigurationManager = _openIdMetadataCache.GetOrAdd(azureBotServiceOpenIdMetadataUrl, key =>
                        {
                            return new ConfigurationManager<OpenIdConnectConfiguration>(azureBotServiceOpenIdMetadataUrl, new OpenIdConnectConfigurationRetriever(), new HttpClient())
                            {
                                AutomaticRefreshInterval = openIdRefreshInterval
                            };
                        });
                    }
                    else
                    {
                        context.Options.TokenValidationParameters.ConfigurationManager = _openIdMetadataCache.GetOrAdd(openIdMetadataUrl, key =>
                        {
                            return new ConfigurationManager<OpenIdConnectConfiguration>(openIdMetadataUrl, new OpenIdConnectConfigurationRetriever(), new HttpClient())
                            {
                                AutomaticRefreshInterval = openIdRefreshInterval
                            };
                        });
                    }

                    await Task.CompletedTask.ConfigureAwait(false);
                },

                OnTokenValidated = context =>
                {
                    // 知らない Entra ID テナントからのアクセスは弾きたい
                    var claims = context.Principal?.Claims;
                    var serviceUrlClaim = claims?.FirstOrDefault(c => c.Type == "serviceurl");

                    if (serviceUrlClaim == null)
                    {
                        context.Fail("Missing serviceurl claim.");
                        logger?.LogWarning("Missing serviceurl claim.");
                        return Task.CompletedTask;
                    }

                    if (!serviceUrlClaim.Value.Contains(tenantId, StringComparison.OrdinalIgnoreCase))
                    {
                        context.Fail($"serviceurl does not contain required tenant ID: {tenantId}");
                        logger?.LogWarning("serviceurl does not contain required tenant ID. serviceurl={0}", serviceUrlClaim.Value);
                        return Task.CompletedTask;
                    }
                    logger?.LogDebug("TOKEN Validated");
                    return Task.CompletedTask;
                },
                OnForbidden = context =>
                {
                    logger?.LogWarning("Forbidden: {m}", context.Result.ToString());
                    return Task.CompletedTask;
                },
                OnAuthenticationFailed = context =>
                {
                    logger?.LogWarning("Auth Failed {m}", context.Exception.ToString());
                    return Task.CompletedTask;
                }
            };
        });
    }

```

以上のコードを実際の環境にデプロイして動かせば期待の挙動ができるはずです。他の方に教えてもらいましたが、Tenant ID チェックには別の方法もありました。Bot/WeatherAgentBot.cs の MessageActivityAsync メソッド内でも以下の様に Tenant ID の取得が可能です。

```csharp

public class WeatherAgentBot : AgentApplication
{
    private WeatherForecastAgent _weatherAgent;
    private Kernel _kernel;

    protected async Task MessageActivityAsync(ITurnContext turnContext, ITurnState turnState, CancellationToken cancellationToken)
    {
        var tenantId = turnContext.Activity.Conversation.TenantId; // add code
        // add validation of tenant ID here


```

こちらを以下のコードを参考にロジックを拡張しても良いはずです。
https://github.com/OfficeDev/microsoft-teams-apps-company-communicator/blob/dcf3b169084d3fff7c1e4c5b68718fb33c3391dd/Source/CompanyCommunicator/Bot/CompanyCommunicatorBotFilterMiddleware.cs#L44


皆様のご参考になれば幸いです。
