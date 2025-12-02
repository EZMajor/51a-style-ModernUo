51alpha Launcher Architecture Specification
Version 1.0 | December 2024
Designed for AI-Assisted Development

Table of Contents

Executive Summary
System Architecture Overview
Discord Application Setup
Launcher Core Components
Web API Integration
Auto-Update System
Authentication Flow
ClassicUO Integration
Configuration Management
UI/UX Specifications
Error Handling & Logging
Security Considerations
Build & Deployment
Testing Strategy
Implementation Phases
Appendices


1. Executive Summary
1.1 Purpose
The 51alpha Launcher is a Windows desktop application that serves as the primary gateway for players to access the 51alpha Ultima Online private server. It provides a unified experience for authentication, client updates, news delivery, and game launching.
1.2 Key Features

Discord OAuth2 authentication with JWT token management
Automatic ClassicUO client updates via CDN
Real-time server status and player count display
News feed with patch notes and announcements
Client configuration management
Seamless ClassicUO launch with authentication passthrough

1.3 Integration Points
SystemIntegration TypePurpose51alpha Web APIREST APIAuthentication, status, news, updatesDiscord OAuth2OAuth2 + WebhooksUser authentication, community linking51alpha WebsiteShared authSingle sign-on, account managementClassicUO ClientProcess launchGame client execution with authModernUO ServerToken validationAuthenticated game connectionsCDN (Cloudflare)HTTPSClient file distribution
1.4 Technology Stack
LayerTechnologyRationaleFrameworkWPF (.NET 10)Modern Windows UI, MVVM supportLanguageC# 13Matches ModernUO codebaseUI ToolkitMaterial Design XAMLProfessional, consistent designHTTP ClientHttpClient + PollyResilient API communicationLocal StorageSQLite + DPAPISecure credential storageLoggingSerilogStructured logging, matches server

2. System Architecture Overview
2.1 High-Level Architecture
┌─────────────────────────────────────────────────────────────────────────┐
│                          51alpha Launcher (WPF)                          │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Login     │  │   News      │  │  Settings   │  │   Update    │    │
│  │   View      │  │   View      │  │   View      │  │   View      │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         │                │                │                │            │
│  ┌──────┴────────────────┴────────────────┴────────────────┴──────┐    │
│  │                        ViewModel Layer                          │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │    │
│  │  │ LoginVM  │ │ NewsVM   │ │SettingsVM│ │ UpdateVM │           │    │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘           │    │
│  └───────┼────────────┼────────────┼────────────┼─────────────────┘    │
│          │            │            │            │                       │
│  ┌───────┴────────────┴────────────┴────────────┴─────────────────┐    │
│  │                        Service Layer                            │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │    │
│  │  │ AuthSvc  │ │ ApiSvc   │ │ UpdateSvc│ │ LaunchSvc│           │    │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘           │    │
│  └───────┼────────────┼────────────┼────────────┼─────────────────┘    │
│          │            │            │            │                       │
│  ┌───────┴────────────┴────────────┴────────────┴─────────────────┐    │
│  │                     Infrastructure Layer                        │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │    │
│  │  │HttpClient│ │ Storage  │ │  Config  │ │  Logger  │           │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│  Discord API  │         │ 51alpha API   │         │  CDN (Files)  │
│  (OAuth2)     │         │ (REST)        │         │  (Updates)    │
└───────────────┘         └───────────────┘         └───────────────┘
2.2 Component Responsibilities
2.2.1 View Layer (XAML)

Pure UI rendering with data binding
No business logic - all logic in ViewModels
Responsive design for various screen sizes

2.2.2 ViewModel Layer

MVVM pattern with INotifyPropertyChanged
Command handling via ICommand implementations
State management and UI orchestration

2.2.3 Service Layer

Business logic encapsulation
API communication via typed clients
Cross-cutting concerns (caching, retry logic)

2.2.4 Infrastructure Layer

HTTP client with retry policies (Polly)
Secure storage (DPAPI encryption)
Configuration management
Structured logging (Serilog)

2.3 Project Structure
51alpha.Launcher/
├── 51alpha.Launcher.sln
├── src/
│   ├── 51alpha.Launcher/                 # Main WPF application
│   │   ├── App.xaml                      # Application entry point
│   │   ├── App.xaml.cs
│   │   ├── Views/                        # XAML views
│   │   │   ├── MainWindow.xaml
│   │   │   ├── LoginView.xaml
│   │   │   ├── NewsView.xaml
│   │   │   ├── SettingsView.xaml
│   │   │   └── UpdateView.xaml
│   │   ├── ViewModels/                   # MVVM ViewModels
│   │   │   ├── MainViewModel.cs
│   │   │   ├── LoginViewModel.cs
│   │   │   ├── NewsViewModel.cs
│   │   │   ├── SettingsViewModel.cs
│   │   │   └── UpdateViewModel.cs
│   │   ├── Models/                       # Data models
│   │   │   ├── UserSession.cs
│   │   │   ├── ServerStatus.cs
│   │   │   ├── NewsItem.cs
│   │   │   ├── UpdateManifest.cs
│   │   │   └── LauncherConfig.cs
│   │   ├── Services/                     # Business logic
│   │   │   ├── IAuthService.cs
│   │   │   ├── AuthService.cs
│   │   │   ├── IApiService.cs
│   │   │   ├── ApiService.cs
│   │   │   ├── IUpdateService.cs
│   │   │   ├── UpdateService.cs
│   │   │   ├── ILaunchService.cs
│   │   │   └── LaunchService.cs
│   │   ├── Infrastructure/               # Cross-cutting
│   │   │   ├── SecureStorage.cs
│   │   │   ├── HttpClientFactory.cs
│   │   │   └── LoggingConfig.cs
│   │   ├── Converters/                   # XAML converters
│   │   ├── Resources/                    # Assets, styles
│   │   │   ├── Styles/
│   │   │   ├── Icons/
│   │   │   └── Themes/
│   │   └── 51alpha.Launcher.csproj
│   │
│   └── 51alpha.Launcher.Shared/          # Shared DTOs/contracts
│       ├── Contracts/
│       │   ├── AuthContracts.cs
│       │   ├── StatusContracts.cs
│       │   └── UpdateContracts.cs
│       └── 51alpha.Launcher.Shared.csproj
│
├── tests/
│   ├── 51alpha.Launcher.Tests/           # Unit tests
│   └── 51alpha.Launcher.Integration/     # Integration tests
│
└── build/
    ├── build.ps1                         # Build script
    ├── publish.ps1                       # Publish/package
    └── installer/                        # WiX/Inno Setup

3. Discord Application Setup
3.1 Creating Discord Application
Before implementing authentication, you must create a Discord application in the Discord Developer Portal.
3.1.1 Step-by-Step Setup

Navigate to https://discord.com/developers/applications
Click 'New Application'
Name it '51alpha' (or your preferred name)
Accept the Developer Terms of Service
Navigate to 'OAuth2' section in the left sidebar
Copy the 'Client ID' - save this securely
Click 'Reset Secret' to generate Client Secret - save this securely

3.1.2 Redirect URIs Configuration
Add the following redirect URIs in the OAuth2 settings:
# For Launcher (localhost callback)
http://localhost:51401/callback

# For Website (production)
https://51alpha.com/auth/discord/callback

# For Website (development)
http://localhost:3000/auth/discord/callback
3.1.3 OAuth2 Scopes Required
ScopePurposeRequiredidentifyAccess user ID, username, avatarYesemailAccess user email (optional)RecommendedguildsCheck server membershipOptionalguilds.members.readCheck roles in 51alpha DiscordOptional
3.1.4 Bot Token (Optional - for Webhooks)
If you want Discord webhook notifications for game events:

Navigate to 'Bot' section
Click 'Add Bot'
Copy the Bot Token - save securely
Enable 'Server Members Intent' if checking roles

3.2 Environment Configuration
Store these secrets securely - NEVER commit to source control:
json// appsettings.secrets.json (gitignored)
{
  "Discord": {
    "ClientId": "YOUR_CLIENT_ID_HERE",
    "ClientSecret": "YOUR_CLIENT_SECRET_HERE",
    "BotToken": "YOUR_BOT_TOKEN_HERE_IF_USING"
  }
}
3.3 Discord OAuth2 Flow Diagram
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Launcher  │     │  Discord    │     │  51alpha    │     │   Browser   │
│   (WPF)     │     │  OAuth2     │     │  API        │     │             │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │                   │
       │  1. User clicks   │                   │                   │
       │     "Login"       │                   │                   │
       │───────────────────┼───────────────────┼──────────────────▶│
       │                   │                   │    2. Open Auth   │
       │                   │                   │       URL         │
       │                   │◀──────────────────┼───────────────────│
       │                   │  3. User logs in  │                   │
       │                   │     & authorizes  │                   │
       │                   │───────────────────┼───────────────────▶│
       │                   │                   │  4. Redirect to   │
       │◀──────────────────┼───────────────────┼───────────────────│
       │  5. Receive code  │                   │    localhost:     │
       │     via callback  │                   │    51401/callback │
       │───────────────────┼───────────────────▶│                   │
       │                   │                   │  6. Exchange code │
       │                   │◀──────────────────│     for tokens    │
       │                   │  7. Return tokens │                   │
       │                   │───────────────────▶│                   │
       │◀──────────────────┼───────────────────│  8. Return JWT    │
       │  9. Store JWT     │                   │     + user info   │
       │     locally       │                   │                   │

4. Launcher Core Components
4.1 Models
4.1.1 UserSession.cs
csharpnamespace _51alpha.Launcher.Models;

public class UserSession
{
    public string UserId { get; init; } = string.Empty;
    public string Username { get; init; } = string.Empty;
    public string Discriminator { get; init; } = string.Empty;
    public string? AvatarUrl { get; init; }
    public string? Email { get; init; }
    
    // JWT tokens
    public string AccessToken { get; set; } = string.Empty;
    public string RefreshToken { get; set; } = string.Empty;
    public DateTime ExpiresAt { get; set; }
    
    // Game-specific
    public string? FactionName { get; init; }
    public int FactionRank { get; init; }
    public string? CharacterName { get; init; }
    
    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
    public bool NeedsRefresh => DateTime.UtcNow >= ExpiresAt.AddMinutes(-5);
    
    public string DisplayName => string.IsNullOrEmpty(Discriminator) || Discriminator == "0"
        ? Username
        : $"{Username}#{Discriminator}";
}
4.1.2 ServerStatus.cs
csharpnamespace _51alpha.Launcher.Models;

public class ServerStatus
{
    public bool IsOnline { get; init; }
    public int PlayerCount { get; init; }
    public int MaxPlayers { get; init; }
    public TimeSpan Uptime { get; init; }
    public DateTime? NextSiegeWindow { get; init; }
    public DateTime? NextEvent { get; init; }
    public string? EventName { get; init; }
    public ServerHealthStatus Health { get; init; }
    public DateTime LastUpdated { get; init; }
}

public enum ServerHealthStatus
{
    Healthy,
    Degraded,
    Maintenance,
    Offline
}
4.1.3 NewsItem.cs
csharpnamespace _51alpha.Launcher.Models;

public class NewsItem
{
    public int Id { get; init; }
    public string Title { get; init; } = string.Empty;
    public string Summary { get; init; } = string.Empty;
    public string? Content { get; init; }
    public string? ImageUrl { get; init; }
    public string Author { get; init; } = string.Empty;
    public DateTime PublishedAt { get; init; }
    public NewsCategory Category { get; init; }
    public string? ExternalUrl { get; init; }
    public bool IsPinned { get; init; }
}

public enum NewsCategory
{
    Announcement,
    PatchNotes,
    Event,
    Maintenance,
    Community
}
4.1.4 UpdateManifest.cs
csharpnamespace _51alpha.Launcher.Models;

public class UpdateManifest
{
    public string CurrentVersion { get; init; } = string.Empty;
    public string MinimumVersion { get; init; } = string.Empty;
    public string? UpdateUrl { get; init; }
    public string? Checksum { get; init; }
    public long? FileSizeBytes { get; init; }
    public bool IsForced { get; init; }
    public string? ReleaseNotes { get; init; }
    public List<PatchFile> Patches { get; init; } = new();
}

public class PatchFile
{
    public string FileName { get; init; } = string.Empty;
    public string RelativePath { get; init; } = string.Empty;
    public string DownloadUrl { get; init; } = string.Empty;
    public string Checksum { get; init; } = string.Empty;
    public long FileSizeBytes { get; init; }
    public PatchAction Action { get; init; }
}

public enum PatchAction
{
    Add,
    Update,
    Delete
}
4.1.5 LauncherConfig.cs
csharpnamespace _51alpha.Launcher.Models;

public class LauncherConfig
{
    // Paths
    public string ClassicUOPath { get; set; } = string.Empty;
    public string ClientDataPath { get; set; } = string.Empty;
    
    // Connection
    public string ServerAddress { get; set; } = "play.51alpha.com";
    public int ServerPort { get; set; } = 2593;
    
    // Display
    public bool StartMinimized { get; set; }
    public bool MinimizeToTray { get; set; } = true;
    public bool CheckUpdatesOnStart { get; set; } = true;
    public bool AutoLaunchAfterUpdate { get; set; }
    
    // ClassicUO Settings Pass-through
    public bool UseHardwareAcceleration { get; set; } = true;
    public string Resolution { get; set; } = "1920x1080";
    public bool Fullscreen { get; set; }
    public int FpsLimit { get; set; } = 60;
    
    // Remember login
    public bool RememberMe { get; set; } = true;
}
4.2 Services
4.2.1 IAuthService Interface
csharpnamespace _51alpha.Launcher.Services;

public interface IAuthService
{
    Task<AuthResult> LoginWithDiscordAsync(CancellationToken ct = default);
    Task<AuthResult> RefreshTokenAsync(CancellationToken ct = default);
    Task LogoutAsync();
    Task<UserSession?> GetCurrentSessionAsync();
    bool HasStoredCredentials { get; }
    
    event EventHandler<UserSession>? SessionChanged;
    event EventHandler? SessionExpired;
}

public class AuthResult
{
    public bool Success { get; init; }
    public UserSession? Session { get; init; }
    public string? ErrorMessage { get; init; }
    public AuthErrorCode? ErrorCode { get; init; }
}

public enum AuthErrorCode
{
    None,
    UserCancelled,
    NetworkError,
    InvalidCredentials,
    TokenExpired,
    ServerError,
    AccountBanned,
    AccountNotLinked
}
4.2.2 AuthService Implementation
csharpnamespace _51alpha.Launcher.Services;

public class AuthService : IAuthService
{
    private readonly IApiService _api;
    private readonly ISecureStorage _storage;
    private readonly ILogger<AuthService> _logger;
    private readonly OAuthCallbackServer _callbackServer;
    
    private const int CallbackPort = 51401;
    private const string CallbackPath = "/callback";
    private UserSession? _currentSession;
    
    public event EventHandler<UserSession>? SessionChanged;
    public event EventHandler? SessionExpired;
    
    public bool HasStoredCredentials => _storage.HasValue("refresh_token");
    
    public AuthService(IApiService api, ISecureStorage storage, 
        ILogger<AuthService> logger)
    {
        _api = api;
        _storage = storage;
        _logger = logger;
        _callbackServer = new OAuthCallbackServer(CallbackPort, CallbackPath);
    }
    
    public async Task<AuthResult> LoginWithDiscordAsync(CancellationToken ct = default)
    {
        try
        {
            // 1. Start local callback server
            var codeTask = _callbackServer.WaitForCodeAsync(ct);
            
            // 2. Generate state for CSRF protection
            var state = Guid.NewGuid().ToString("N");
            _storage.Set("oauth_state", state);
            
            // 3. Build Discord auth URL
            var authUrl = BuildDiscordAuthUrl(state);
            
            // 4. Open browser
            Process.Start(new ProcessStartInfo(authUrl) { UseShellExecute = true });
            
            // 5. Wait for callback (with timeout)
            using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
            cts.CancelAfter(TimeSpan.FromMinutes(5));
            
            var (code, returnedState) = await codeTask;
            
            // 6. Validate state
            var savedState = _storage.Get("oauth_state");
            if (returnedState != savedState)
            {
                return new AuthResult 
                { 
                    Success = false, 
                    ErrorCode = AuthErrorCode.InvalidCredentials,
                    ErrorMessage = "OAuth state mismatch - possible CSRF attack"
                };
            }
            
            // 7. Exchange code for tokens via our API
            var tokenResponse = await _api.ExchangeDiscordCodeAsync(code, ct);
            
            if (!tokenResponse.Success)
            {
                return new AuthResult
                {
                    Success = false,
                    ErrorCode = tokenResponse.ErrorCode,
                    ErrorMessage = tokenResponse.ErrorMessage
                };
            }
            
            // 8. Store tokens securely
            _currentSession = tokenResponse.Session;
            await StoreSessionAsync(_currentSession);
            
            SessionChanged?.Invoke(this, _currentSession);
            
            return new AuthResult { Success = true, Session = _currentSession };
        }
        catch (OperationCanceledException)
        {
            return new AuthResult 
            { 
                Success = false, 
                ErrorCode = AuthErrorCode.UserCancelled 
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Discord login failed");
            return new AuthResult 
            { 
                Success = false, 
                ErrorCode = AuthErrorCode.NetworkError,
                ErrorMessage = ex.Message 
            };
        }
    }
    
    private string BuildDiscordAuthUrl(string state)
    {
        var clientId = Configuration.DiscordClientId;
        var redirectUri = Uri.EscapeDataString($"http://localhost:{CallbackPort}{CallbackPath}");
        var scopes = Uri.EscapeDataString("identify email");
        
        return $"https://discord.com/oauth2/authorize" +
               $"?client_id={clientId}" +
               $"&redirect_uri={redirectUri}" +
               $"&response_type=code" +
               $"&scope={scopes}" +
               $"&state={state}";
    }
}
4.2.3 OAuthCallbackServer
csharpnamespace _51alpha.Launcher.Services;

/// <summary>
/// Lightweight HTTP server to receive OAuth callback on localhost
/// </summary>
public class OAuthCallbackServer : IDisposable
{
    private readonly HttpListener _listener;
    private readonly string _callbackPath;
    
    public OAuthCallbackServer(int port, string path)
    {
        _callbackPath = path;
        _listener = new HttpListener();
        _listener.Prefixes.Add($"http://localhost:{port}/");
    }
    
    public async Task<(string Code, string State)> WaitForCodeAsync(
        CancellationToken ct = default)
    {
        _listener.Start();
        
        try
        {
            while (!ct.IsCancellationRequested)
            {
                var context = await _listener.GetContextAsync().WaitAsync(ct);
                
                var request = context.Request;
                var response = context.Response;
                
                if (request.Url?.LocalPath == _callbackPath)
                {
                    var query = request.QueryString;
                    var code = query["code"];
                    var state = query["state"];
                    var error = query["error"];
                    
                    // Send response page to browser
                    var html = error != null ? GetErrorHtml(error) : GetSuccessHtml();
                    
                    var buffer = Encoding.UTF8.GetBytes(html);
                    response.ContentType = "text/html";
                    response.ContentLength64 = buffer.Length;
                    await response.OutputStream.WriteAsync(buffer, ct);
                    response.Close();
                    
                    if (error != null)
                        throw new AuthException($"Discord auth error: {error}");
                    
                    if (string.IsNullOrEmpty(code))
                        throw new AuthException("No authorization code received");
                    
                    return (code, state ?? string.Empty);
                }
                
                response.StatusCode = 404;
                response.Close();
            }
            
            throw new OperationCanceledException();
        }
        finally
        {
            _listener.Stop();
        }
    }
    
    private static string GetSuccessHtml() => """
        <!DOCTYPE html>
        <html>
        <head>
            <title>51alpha - Login Successful</title>
            <style>
                body { 
                    font-family: Arial, sans-serif; 
                    display: flex; 
                    justify-content: center; 
                    align-items: center; 
                    height: 100vh; 
                    margin: 0;
                    background: linear-gradient(135deg, #1a365d 0%, #2c5282 100%);
                    color: white;
                }
                .container { text-align: center; }
                h1 { font-size: 2em; margin-bottom: 0.5em; }
                p { opacity: 0.8; }
            </style>
        </head>
        <body>
            <div class="container">
                <h1>✓ Login Successful!</h1>
                <p>You can close this window and return to the launcher.</p>
            </div>
        </body>
        </html>
        """;
    
    public void Dispose() => _listener.Close();
}
4.2.4 IApiService Interface
csharpnamespace _51alpha.Launcher.Services;

public interface IApiService
{
    // Authentication
    Task<TokenResponse> ExchangeDiscordCodeAsync(string code, CancellationToken ct);
    Task<TokenResponse> RefreshTokenAsync(string refreshToken, CancellationToken ct);
    
    // Status
    Task<ServerStatus> GetServerStatusAsync(CancellationToken ct);
    
    // News
    Task<List<NewsItem>> GetNewsAsync(int page = 1, int limit = 10, CancellationToken ct = default);
    Task<NewsItem?> GetNewsItemAsync(int id, CancellationToken ct);
    
    // Updates
    Task<UpdateManifest> GetUpdateManifestAsync(string currentVersion, CancellationToken ct);
    Task<Stream> DownloadFileAsync(string url, IProgress<double>? progress, CancellationToken ct);
    
    // Player (authenticated)
    Task<PlayerInfo> GetCurrentPlayerAsync(CancellationToken ct);
}
4.2.5 IUpdateService Interface
csharpnamespace _51alpha.Launcher.Services;

public interface IUpdateService
{
    Task<UpdateCheckResult> CheckForUpdatesAsync(CancellationToken ct = default);
    Task<UpdateResult> ApplyUpdatesAsync(UpdateManifest manifest, 
        IProgress<UpdateProgress>? progress = null, CancellationToken ct = default);
    Task<bool> VerifyInstallationAsync(CancellationToken ct = default);
    string GetCurrentVersion();
}

public class UpdateCheckResult
{
    public bool UpdateAvailable { get; init; }
    public bool UpdateRequired { get; init; }
    public UpdateManifest? Manifest { get; init; }
    public string CurrentVersion { get; init; } = string.Empty;
    public string? LatestVersion { get; init; }
}

public class UpdateProgress
{
    public UpdateStage Stage { get; init; }
    public string CurrentFile { get; init; } = string.Empty;
    public int FilesCompleted { get; init; }
    public int TotalFiles { get; init; }
    public long BytesDownloaded { get; init; }
    public long TotalBytes { get; init; }
    public double PercentComplete => TotalBytes > 0 
        ? (double)BytesDownloaded / TotalBytes * 100 : 0;
}

public enum UpdateStage
{
    Checking,
    Downloading,
    Verifying,
    Applying,
    CleaningUp,
    Complete,
    Failed
}
4.2.6 ILaunchService Interface
csharpnamespace _51alpha.Launcher.Services;

public interface ILaunchService
{
    Task<LaunchResult> LaunchGameAsync(UserSession session, CancellationToken ct = default);
    bool IsGameRunning { get; }
    Task KillGameAsync();
    
    event EventHandler? GameStarted;
    event EventHandler<int>? GameExited;
}

public class LaunchResult
{
    public bool Success { get; init; }
    public int? ProcessId { get; init; }
    public string? ErrorMessage { get; init; }
    public LaunchErrorCode? ErrorCode { get; init; }
}

public enum LaunchErrorCode
{
    None,
    ClientNotFound,
    ClientOutdated,
    InvalidSession,
    ProcessStartFailed,
    AlreadyRunning
}

5. Web API Integration
5.1 API Base Configuration
csharpnamespace _51alpha.Launcher.Infrastructure;

public static class ApiConfiguration
{
    public const string BaseUrl = "https://api.51alpha.com";
    public const string CdnUrl = "https://cdn.51alpha.com";
    
    public static class Endpoints
    {
        // Auth
        public const string DiscordCallback = "/api/auth/discord";
        public const string RefreshToken = "/api/auth/refresh";
        public const string Logout = "/api/auth/logout";
        
        // Status
        public const string ServerStatus = "/api/status";
        
        // News
        public const string News = "/api/news";
        
        // Updates (Launcher)
        public const string LauncherVersion = "/api/launcher/version";
        public const string LauncherPatches = "/api/launcher/patches";
        
        // Player
        public const string Me = "/api/me";
    }
}
5.2 API Request/Response Contracts
5.2.1 Authentication Contracts
csharp// POST /api/auth/discord
public class DiscordAuthRequest
{
    public string Code { get; init; } = string.Empty;
    public string RedirectUri { get; init; } = string.Empty;
}

public class AuthResponse
{
    public bool Success { get; init; }
    public string? AccessToken { get; init; }
    public string? RefreshToken { get; init; }
    public int ExpiresIn { get; init; }
    public UserInfo? User { get; init; }
    public string? Error { get; init; }
    public string? ErrorDescription { get; init; }
}

public class UserInfo
{
    public string Id { get; init; } = string.Empty;
    public string Username { get; init; } = string.Empty;
    public string? Discriminator { get; init; }
    public string? Avatar { get; init; }
    public string? Email { get; init; }
    public bool EmailVerified { get; init; }
    
    // 51alpha-specific
    public string? LinkedCharacter { get; init; }
    public string? Faction { get; init; }
    public int? FactionRank { get; init; }
    public List<string> Roles { get; init; } = new();
}
5.2.2 Status Contracts
csharp// GET /api/status
public class StatusResponse
{
    public bool Online { get; init; }
    public int Players { get; init; }
    public int MaxPlayers { get; init; }
    public double UptimeHours { get; init; }
    public string? NextSiegeWindow { get; init; }  // ISO 8601
    public string? NextEvent { get; init; }        // ISO 8601
    public string? EventName { get; init; }
    public string Health { get; init; } = "healthy";
    public string ServerVersion { get; init; } = string.Empty;
}
5.2.3 News Contracts
csharp// GET /api/news?page=1&limit=10
public class NewsResponse
{
    public List<NewsItemDto> Items { get; init; } = new();
    public int TotalCount { get; init; }
    public int Page { get; init; }
    public int PageSize { get; init; }
    public bool HasMore { get; init; }
}

public class NewsItemDto
{
    public int Id { get; init; }
    public string Title { get; init; } = string.Empty;
    public string Summary { get; init; } = string.Empty;
    public string? Content { get; init; }
    public string? ImageUrl { get; init; }
    public string Author { get; init; } = string.Empty;
    public string PublishedAt { get; init; } = string.Empty;  // ISO 8601
    public string Category { get; init; } = string.Empty;
    public string? ExternalUrl { get; init; }
    public bool IsPinned { get; init; }
}
5.2.4 Update Contracts
csharp// GET /api/launcher/version
public class VersionResponse
{
    public string CurrentVersion { get; init; } = string.Empty;
    public string MinimumVersion { get; init; } = string.Empty;
    public string UpdateUrl { get; init; } = string.Empty;
    public string Checksum { get; init; } = string.Empty;
    public long FileSizeBytes { get; init; }
    public bool IsForced { get; init; }
    public string? ReleaseNotes { get; init; }
}

// GET /api/launcher/patches?from=1.2.2
public class PatchResponse
{
    public string FromVersion { get; init; } = string.Empty;
    public string ToVersion { get; init; } = string.Empty;
    public List<PatchFileDto> Patches { get; init; } = new();
    public long TotalSizeBytes { get; init; }
}

public class PatchFileDto
{
    public string FileName { get; init; } = string.Empty;
    public string RelativePath { get; init; } = string.Empty;
    public string Url { get; init; } = string.Empty;
    public string Checksum { get; init; } = string.Empty;
    public long SizeBytes { get; init; }
    public string Action { get; init; } = "update";  // add, update, delete
}
5.3 HTTP Client Configuration
csharpnamespace _51alpha.Launcher.Infrastructure;

public static class HttpClientConfiguration
{
    public static IServiceCollection AddApiClients(this IServiceCollection services)
    {
        services.AddHttpClient<IApiService, ApiService>(client =>
        {
            client.BaseAddress = new Uri(ApiConfiguration.BaseUrl);
            client.DefaultRequestHeaders.Accept.Add(
                new MediaTypeWithQualityHeaderValue("application/json"));
            client.DefaultRequestHeaders.UserAgent.ParseAdd(
                $"51alpha-Launcher/{GetVersion()}");
            client.Timeout = TimeSpan.FromSeconds(30);
        })
        .AddPolicyHandler(GetRetryPolicy())
        .AddPolicyHandler(GetCircuitBreakerPolicy());
        
        services.AddHttpClient("CDN", client =>
        {
            client.BaseAddress = new Uri(ApiConfiguration.CdnUrl);
            client.Timeout = TimeSpan.FromMinutes(10);
        });
        
        return services;
    }
    
    private static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .Or<TimeoutRejectedException>()
            .WaitAndRetryAsync(3, retryAttempt =>
                TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
    }
    
    private static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
    }
}

6. Auto-Update System
6.1 Update Strategy
The launcher uses a differential patching system to minimize download sizes. Full downloads are available as fallback.
6.1.1 Update Decision Flow
┌─────────────────┐
│  Launcher Start │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Check Version  │──▶ GET /api/launcher/version
└────────┬────────┘
         │
         ▼
    ┌────────────┐
    │ Update     │───No──▶ Continue to login
    │ Available? │
    └─────┬──────┘
          │Yes
          ▼
    ┌────────────┐
    │ Forced     │───Yes──▶ Must update, disable "Skip"
    │ Update?    │
    └─────┬──────┘
          │No
          ▼
┌─────────────────┐
│ Show update UI  │──▶ User can choose Update/Skip
└────────┬────────┘
         │Update
         ▼
    ┌────────────┐
    │ Patches    │───Yes──▶ Download patches only
    │ Available? │
    └─────┬──────┘
          │No
          ▼
┌─────────────────┐
│ Full download   │──▶ Download complete package
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Verify checksum │
└────────┬────────┘
         │
         ▼
    ┌────────────┐
    │ Valid?     │───No──▶ Retry or error
    └─────┬──────┘
          │Yes
          ▼
┌─────────────────┐
│ Apply updates   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Continue        │
└─────────────────┘
6.2 UpdateService Implementation
csharpnamespace _51alpha.Launcher.Services;

public class UpdateService : IUpdateService
{
    private readonly IApiService _api;
    private readonly HttpClient _cdnClient;
    private readonly ILogger<UpdateService> _logger;
    private readonly string _installPath;
    private readonly string _tempPath;
    
    private static readonly string VersionFile = "version.txt";
    
    public string GetCurrentVersion()
    {
        var versionPath = Path.Combine(_installPath, VersionFile);
        return File.Exists(versionPath) 
            ? File.ReadAllText(versionPath).Trim() 
            : "0.0.0";
    }
    
    public async Task<UpdateCheckResult> CheckForUpdatesAsync(CancellationToken ct)
    {
        var current = GetCurrentVersion();
        var manifest = await _api.GetUpdateManifestAsync(current, ct);
        
        var latestVersion = Version.Parse(manifest.CurrentVersion);
        var currentVersion = Version.Parse(current);
        var minVersion = Version.Parse(manifest.MinimumVersion);
        
        return new UpdateCheckResult
        {
            UpdateAvailable = latestVersion > currentVersion,
            UpdateRequired = currentVersion < minVersion,
            Manifest = latestVersion > currentVersion ? manifest : null,
            CurrentVersion = current,
            LatestVersion = manifest.CurrentVersion
        };
    }
    
    public async Task<UpdateResult> ApplyUpdatesAsync(UpdateManifest manifest,
        IProgress<UpdateProgress>? progress, CancellationToken ct)
    {
        Directory.CreateDirectory(_tempPath);
        
        try
        {
            var files = manifest.Patches.Any() 
                ? manifest.Patches 
                : new List<PatchFile> { CreateFullDownloadPatch(manifest) };
            
            var totalBytes = files.Sum(f => f.FileSizeBytes);
            long downloadedBytes = 0;
            
            for (int i = 0; i < files.Count; i++)
            {
                var file = files[i];
                var tempFile = Path.Combine(_tempPath, file.FileName);
                
                progress?.Report(new UpdateProgress
                {
                    Stage = UpdateStage.Downloading,
                    CurrentFile = file.FileName,
                    FilesCompleted = i,
                    TotalFiles = files.Count,
                    BytesDownloaded = downloadedBytes,
                    TotalBytes = totalBytes
                });
                
                await DownloadFileWithProgressAsync(file.DownloadUrl, tempFile,
                    new Progress<long>(bytes =>
                    {
                        progress?.Report(new UpdateProgress
                        {
                            Stage = UpdateStage.Downloading,
                            CurrentFile = file.FileName,
                            FilesCompleted = i,
                            TotalFiles = files.Count,
                            BytesDownloaded = downloadedBytes + bytes,
                            TotalBytes = totalBytes
                        });
                    }), ct);
                
                // Verify checksum
                progress?.Report(new UpdateProgress 
                { 
                    Stage = UpdateStage.Verifying, 
                    CurrentFile = file.FileName 
                });
                
                var hash = await ComputeFileHashAsync(tempFile, ct);
                if (!hash.Equals(file.Checksum, StringComparison.OrdinalIgnoreCase))
                {
                    throw new UpdateException($"Checksum mismatch for {file.FileName}");
                }
                
                downloadedBytes += file.FileSizeBytes;
            }
            
            // Apply phase
            progress?.Report(new UpdateProgress { Stage = UpdateStage.Applying });
            
            foreach (var file in files)
            {
                var tempFile = Path.Combine(_tempPath, file.FileName);
                var targetPath = Path.Combine(_installPath, file.RelativePath);
                
                switch (file.Action)
                {
                    case PatchAction.Add:
                    case PatchAction.Update:
                        Directory.CreateDirectory(Path.GetDirectoryName(targetPath)!);
                        File.Copy(tempFile, targetPath, overwrite: true);
                        break;
                    case PatchAction.Delete:
                        if (File.Exists(targetPath)) File.Delete(targetPath);
                        break;
                }
            }
            
            File.WriteAllText(Path.Combine(_installPath, VersionFile), 
                manifest.CurrentVersion);
            
            progress?.Report(new UpdateProgress { Stage = UpdateStage.CleaningUp });
            Directory.Delete(_tempPath, recursive: true);
            
            progress?.Report(new UpdateProgress { Stage = UpdateStage.Complete });
            
            return new UpdateResult { Success = true };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Update failed");
            progress?.Report(new UpdateProgress { Stage = UpdateStage.Failed });
            return new UpdateResult { Success = false, ErrorMessage = ex.Message };
        }
    }
    
    private static async Task<string> ComputeFileHashAsync(string path, CancellationToken ct)
    {
        using var sha256 = SHA256.Create();
        await using var stream = File.OpenRead(path);
        var hash = await sha256.ComputeHashAsync(stream, ct);
        return Convert.ToHexString(hash);
    }
}

7. Authentication Flow
7.1 Complete Auth Sequence
┌──────────────────────────────────────────────────────────────────────────────┐
│                         AUTHENTICATION FLOW                                  │
└──────────────────────────────────────────────────────────────────────────────┘

1. LAUNCHER STARTUP
   ┌─────────────┐
   │ App.OnStart │
   └──────┬──────┘
          │
          ▼
   ┌─────────────────┐
   │ Check stored    │
   │ refresh token   │
   └────────┬────────┘
            │
      ┌─────┴─────┐
      │  Exists?  │
      └─────┬─────┘
          Yes│      │No
             ▼      ▼
   ┌──────────────┐  ┌──────────────┐
   │ Try refresh  │  │ Show login   │
   │ token        │  │ screen       │
   └──────┬───────┘  └──────────────┘
          │
     ┌────┴────┐
     │Success? │
     └────┬────┘
        Yes│     │No
           ▼     ▼
   ┌────────────┐  ┌────────────┐
   │ Auto-login │  │ Clear old  │
   │ complete   │  │ tokens,    │
   └────────────┘  │ show login │
                   └────────────┘
7.2 Token Management
7.2.1 Token Storage (DPAPI)
csharpnamespace _51alpha.Launcher.Infrastructure;

public interface ISecureStorage
{
    void Set(string key, string value);
    string? Get(string key);
    void Remove(string key);
    bool HasValue(string key);
    void Clear();
}

public class SecureStorage : ISecureStorage
{
    private readonly string _storagePath;
    private readonly ILogger<SecureStorage> _logger;
    
    public SecureStorage(ILogger<SecureStorage> logger)
    {
        _logger = logger;
        _storagePath = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
            "51alpha", "Launcher", "secure.dat");
        Directory.CreateDirectory(Path.GetDirectoryName(_storagePath)!);
    }
    
    public void Set(string key, string value)
    {
        var data = LoadData();
        data[key] = value;
        SaveData(data);
    }
    
    public string? Get(string key)
    {
        var data = LoadData();
        return data.TryGetValue(key, out var value) ? value : null;
    }
    
    private Dictionary<string, string> LoadData()
    {
        if (!File.Exists(_storagePath))
            return new Dictionary<string, string>();
        
        try
        {
            var encrypted = File.ReadAllBytes(_storagePath);
            var decrypted = ProtectedData.Unprotect(encrypted, null, 
                DataProtectionScope.CurrentUser);
            var json = Encoding.UTF8.GetString(decrypted);
            return JsonSerializer.Deserialize<Dictionary<string, string>>(json) 
                ?? new Dictionary<string, string>();
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Failed to load secure storage, resetting");
            return new Dictionary<string, string>();
        }
    }
    
    private void SaveData(Dictionary<string, string> data)
    {
        var json = JsonSerializer.Serialize(data);
        var bytes = Encoding.UTF8.GetBytes(json);
        var encrypted = ProtectedData.Protect(bytes, null, 
            DataProtectionScope.CurrentUser);
        File.WriteAllBytes(_storagePath, encrypted);
    }
}

8. ClassicUO Integration
8.1 Launch Configuration
ClassicUO accepts command-line arguments and configuration via settings.json.
8.1.1 ClassicUO Command Line Arguments
ArgumentDescriptionExample-ipServer IP address-ip play.51alpha.com-portServer port-port 2593-usernamePre-fill username-username MyAccount-passwordAuto-login password-password ***-autologinSkip login screen-autologin-fastconnectSkip intro animation-fastconnect-pluginsLoad plugins-plugins razorce.dll-settingsCustom settings file-settings 51alpha.json
8.2 LaunchService Implementation
csharpnamespace _51alpha.Launcher.Services;

public class LaunchService : ILaunchService
{
    private readonly LauncherConfig _config;
    private readonly ILogger<LaunchService> _logger;
    private Process? _gameProcess;
    
    public event EventHandler? GameStarted;
    public event EventHandler<int>? GameExited;
    
    public bool IsGameRunning => _gameProcess?.HasExited == false;
    
    public async Task<LaunchResult> LaunchGameAsync(UserSession session, 
        CancellationToken ct = default)
    {
        if (IsGameRunning)
        {
            return new LaunchResult
            {
                Success = false,
                ErrorCode = LaunchErrorCode.AlreadyRunning,
                ErrorMessage = "Game is already running"
            };
        }
        
        var clientPath = Path.Combine(_config.ClassicUOPath, "ClassicUO.exe");
        if (!File.Exists(clientPath))
        {
            return new LaunchResult
            {
                Success = false,
                ErrorCode = LaunchErrorCode.ClientNotFound,
                ErrorMessage = $"Client not found at {clientPath}"
            };
        }
        
        try
        {
            var settingsPath = await GenerateSettingsFileAsync(session, ct);
            var args = BuildArguments(settingsPath, session);
            
            _logger.LogInformation("Launching ClassicUO: {Path} {Args}", clientPath, args);
            
            var startInfo = new ProcessStartInfo
            {
                FileName = clientPath,
                Arguments = args,
                WorkingDirectory = _config.ClassicUOPath,
                UseShellExecute = false
            };
            
            // Pass auth token via environment variable
            startInfo.EnvironmentVariables["51ALPHA_TOKEN"] = session.AccessToken;
            
            _gameProcess = Process.Start(startInfo);
            
            if (_gameProcess == null)
            {
                return new LaunchResult
                {
                    Success = false,
                    ErrorCode = LaunchErrorCode.ProcessStartFailed,
                    ErrorMessage = "Failed to start game process"
                };
            }
            
            _gameProcess.EnableRaisingEvents = true;
            _gameProcess.Exited += OnGameExited;
            
            GameStarted?.Invoke(this, EventArgs.Empty);
            
            return new LaunchResult { Success = true, ProcessId = _gameProcess.Id };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to launch game");
            return new LaunchResult
            {
                Success = false,
                ErrorCode = LaunchErrorCode.ProcessStartFailed,
                ErrorMessage = ex.Message
            };
        }
    }
    
    private string BuildArguments(string settingsPath, UserSession session)
    {
        var sb = new StringBuilder();
        sb.Append($"-settings \"{settingsPath}\" ");
        sb.Append($"-ip {_config.ServerAddress} ");
        sb.Append($"-port {_config.ServerPort} ");
        sb.Append("-fastconnect ");
        
        if (!string.IsNullOrEmpty(session.Username))
            sb.Append($"-username \"{session.Username}\" ");
        
        return sb.ToString().Trim();
    }
}

9. Configuration Management
9.1 Configuration Locations
Config TypeLocationPurposeApp Settings%LOCALAPPDATA%\51alpha\Launcher\appsettings.jsonGeneral launcher configUser Prefs%LOCALAPPDATA%\51alpha\Launcher\preferences.jsonUI preferencesSecure Data%LOCALAPPDATA%\51alpha\Launcher\secure.datEncrypted tokensClient Settings{ClassicUO}\settings-51alpha.jsonClassicUO game settingsLog Files%LOCALAPPDATA%\51alpha\Launcher\logs\Application logs
9.2 Dependency Injection Setup
csharpnamespace _51alpha.Launcher;

public partial class App : Application
{
    private IHost? _host;
    
    protected override async void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        
        _host = Host.CreateDefaultBuilder()
            .ConfigureAppConfiguration((context, config) =>
            {
                config.SetBasePath(AppContext.BaseDirectory);
                config.AddJsonFile("appsettings.json", optional: false);
                config.AddJsonFile("appsettings.local.json", optional: true);
                config.AddEnvironmentVariables("51ALPHA_");
            })
            .ConfigureServices((context, services) =>
            {
                // Configuration
                services.Configure<AppSettings>(context.Configuration);
                services.AddSingleton(sp => 
                    sp.GetRequiredService<IOptions<AppSettings>>().Value);
                
                // Infrastructure
                services.AddSingleton<ISecureStorage, SecureStorage>();
                services.AddApiClients();
                
                // Services
                services.AddSingleton<IAuthService, AuthService>();
                services.AddSingleton<IUpdateService, UpdateService>();
                services.AddSingleton<ILaunchService, LaunchService>();
                
                // ViewModels
                services.AddTransient<MainViewModel>();
                services.AddTransient<LoginViewModel>();
                
                // Views
                services.AddTransient<MainWindow>();
                
                // Logging
                services.AddLogging(builder =>
                {
                    builder.AddSerilog(new LoggerConfiguration()
                        .MinimumLevel.Information()
                        .WriteTo.File(Path.Combine(GetLogPath(), "launcher-.log"),
                            rollingInterval: RollingInterval.Day,
                            retainedFileCountLimit: 7)
                        .WriteTo.Debug()
                        .CreateLogger());
                });
            })
            .Build();
        
        await _host.StartAsync();
        
        var mainWindow = _host.Services.GetRequiredService<MainWindow>();
        mainWindow.Show();
    }
}

10. UI/UX Specifications
10.1 Visual Design
10.1.1 Color Palette
Color NameHex CodeUsagePrimary Dark#1a365dHeaders, primary buttonsPrimary#2c5282Secondary elementsPrimary Light#4299e1Hover states, linksAccent#ed8936Calls to action, notificationsSuccess#48bb78Success states, online statusWarning#ecc94bWarnings, pending statesError#f56565Errors, offline statusBackground Dark#1a202cMain backgroundBackground#2d3748Card backgroundsText Primary#f7fafcPrimary textText Secondary#a0aec0Secondary text
10.2 Window Layout
┌───────────────────────────────────────────────────────────────────────────┐
│  ─ □ ✕  51alpha Launcher                                                  │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                        [51alpha Logo]                               │  │
│  │                                                                     │  │
│  │                     Welcome, PlayerName                             │  │
│  │                     True Britannians - Rank 5                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  SERVER STATUS                                          ● ONLINE   │  │
│  │  ─────────────────────────────────────────────────────────────────  │  │
│  │  Players: 127 / 500          Uptime: 3d 14h                        │  │
│  │  Next Siege: Today 8:00 PM   Event: Double XP Weekend              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  NEWS                                                               │  │
│  │  ─────────────────────────────────────────────────────────────────  │  │
│  │  📌 Patch 1.2.5 Released                             2 hours ago   │  │
│  │     Bug fixes and performance improvements...                       │  │
│  │                                                                     │  │
│  │  🎮 Weekend Event: Double XP                        Yesterday      │  │
│  │     Earn double talisman XP this weekend...                        │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    ┌─────────────────────┐                          │  │
│  │                    │    ▶  PLAY NOW     │                          │  │
│  │                    └─────────────────────┘                          │  │
│  │                                                                     │  │
│  │    [⚙ Settings]     [🔄 Check Updates]     [🔗 Website]           │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  Version 1.2.0                                     Logged in as Player   │
└───────────────────────────────────────────────────────────────────────────┘

11. Error Handling & Logging
11.1 Error Categories
CategoryExamplesUser MessageActionNetworkAPI timeout, no internetCould not connect to serverRetry with exponential backoffAuthInvalid token, expired sessionPlease log in againClear tokens, show loginUpdateDownload failed, checksum mismatchUpdate failed - please retryOffer retry or skipLaunchClient not found, already runningCould not launch gameShow error details, offer fixConfigInvalid settings, missing pathsConfiguration errorReset to defaults, show settings
11.2 Logging Configuration
csharpLog.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithProperty("Application", "51alpha.Launcher")
    .WriteTo.File(
        path: Path.Combine(logPath, "launcher-.log"),
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 7,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u3}] {Message:lj}{NewLine}{Exception}")
    .WriteTo.Debug()
    .CreateLogger();

12. Security Considerations
12.1 Token Security

Access tokens stored encrypted via Windows DPAPI
Tokens scoped to current Windows user only
Refresh tokens rotated on each use
Short-lived access tokens (1 hour default)

12.2 OAuth Security

CSRF protection via state parameter validation
Code exchange happens server-side (client secret never in launcher)
Localhost callback limits attack surface
Callback server only listens during active auth flow

12.3 Communication Security

All API calls over HTTPS with TLS 1.2+
Certificate pinning for production API endpoints
CDN downloads verified via SHA-256 checksums

12.4 Update Security

All update manifests signed by server
Each file verified via SHA-256 before application
Rollback capability if verification fails
Minimum version enforcement prevents downgrade attacks


13. Build & Deployment
13.1 Build Configuration
xml<!-- 51alpha.Launcher.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net10.0-windows</TargetFramework>
    <UseWPF>true</UseWPF>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <ApplicationIcon>Resources\Icons\launcher.ico</ApplicationIcon>
    <AssemblyName>51alpha.Launcher</AssemblyName>
    <Version>1.0.0</Version>
    <PublishSingleFile>true</PublishSingleFile>
    <SelfContained>true</SelfContained>
    <RuntimeIdentifier>win-x64</RuntimeIdentifier>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="CommunityToolkit.Mvvm" Version="10.2.2" />
    <PackageReference Include="MaterialDesignThemes" Version="4.9.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="10.0.0" />
    <PackageReference Include="Microsoft.Extensions.Http.Polly" Version="10.0.0" />
    <PackageReference Include="Serilog.Extensions.Hosting" Version="10.0.0" />
    <PackageReference Include="Serilog.Sinks.File" Version="5.0.0" />
  </ItemGroup>
</Project>

14. Testing Strategy
14.1 Test Coverage Targets
ComponentTargetPriorityAuthService90%CriticalUpdateService85%CriticalApiService80%HighLaunchService75%HighViewModels70%MediumInfrastructure60%Medium

15. Implementation Phases
15.1 Phase 1: Foundation (Week 1)

Create solution structure and projects
Set up dependency injection and configuration
Implement SecureStorage with DPAPI
Create base ViewModels and MVVM infrastructure
Set up Serilog logging

Acceptance Criteria: App launches, DI works, logs written to file
15.2 Phase 2: Authentication (Week 2)

Create Discord application in developer portal
Implement OAuthCallbackServer
Implement AuthService with Discord flow
Create LoginViewModel and LoginView
Implement token storage and refresh

Acceptance Criteria: Can log in via Discord, session persists after restart
15.3 Phase 3: API Integration (Week 2-3)

Implement IApiService with typed HTTP client
Add Polly retry/circuit breaker policies
Create server status polling
Implement news feed fetching
Build main dashboard UI

Acceptance Criteria: Shows real-time server status and news
15.4 Phase 4: Update System (Week 3)

Implement UpdateService with version checking
Add differential patch downloading
Implement SHA-256 verification
Create update progress UI
Handle forced updates

Acceptance Criteria: Can detect, download, verify, and apply updates
15.5 Phase 5: Game Launch (Week 4)

Implement LaunchService
Generate ClassicUO settings file
Pass auth token to game client
Monitor game process lifecycle
Create settings UI

Acceptance Criteria: Can launch ClassicUO with authenticated session
15.6 Phase 6: Polish & Release (Week 4)

UI polish and Material Design theming
Error handling improvements
Create installer with Inno Setup
Set up CI/CD pipeline
Documentation and README

Acceptance Criteria: Production-ready installer, CI builds passing

16. Appendices
A. NuGet Package References
# Core
Microsoft.Extensions.Hosting 8.0.0
Microsoft.Extensions.Http.Polly 8.0.0
CommunityToolkit.Mvvm 8.2.2

# UI
MaterialDesignThemes 4.9.0
MaterialDesignColors 2.1.4

# Logging
Serilog.Extensions.Hosting 8.0.0
Serilog.Sinks.File 5.0.0

# HTTP
Polly.Extensions.Http 3.0.0

# Storage
System.Security.Cryptography.ProtectedData 8.0.0

# Testing
xunit 2.6.2
Moq 4.20.70
FluentAssertions 6.12.0
B. Environment Variables
VariableDescriptionRequired51ALPHA_API_URLOverride API base URLNo51ALPHA_CDN_URLOverride CDN base URLNo51ALPHA_DISCORD_CLIENT_IDDiscord OAuth client IDBuild only51ALPHA_LOG_LEVELMinimum log levelNo51ALPHA_SKIP_UPDATESkip update check (dev)No
C. API Error Codes
CodeMeaningLauncher ActionAUTH001Invalid or expired tokenRedirect to loginAUTH002Account bannedShow ban message, disable playAUTH003Account not linkedPrompt to link in-gameUPDATE001Client too oldForce updateUPDATE002Download failedRetry with backoffSERVER001Maintenance modeShow maintenance messageSERVER002Server fullShow queue or retry
D. Glossary
TermDefinitionClassicUOOpen-source Ultima Online client used by 51alphaDPAPIWindows Data Protection API for encrypting local dataJWTJSON Web Token used for authenticationModernUOModern C# Ultima Online server emulatorOAuth2Authorization framework used by DiscordPolly.NET resilience library for retry/circuit breakerWPFWindows Presentation Foundation UI framework

End of Document
51alpha Launcher Architecture Specification v1.0
Prepared for AI-Assisted Development