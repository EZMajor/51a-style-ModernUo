51alpha Website Architecture Specification
Version 1.0 | December 2024
Web API • Player Portal • Admin Panel
Designed for AI-Assisted Development

Table of Contents

Executive Summary
System Architecture Overview
Technology Stack
Database Schema
Authentication System
API Endpoints
Admin Panel
Public Website
Game Server Integration
CDN & File Management
Real-Time Features
Security Implementation
Deployment & Infrastructure
Monitoring & Observability
Testing Strategy
Implementation Phases
Appendices


1. Executive Summary
1.1 Purpose
The 51alpha Website serves as the central hub connecting players, the game launcher, and the ModernUO game server. It provides the Web API that powers the launcher, a public-facing website for community engagement, and an administrative panel for server management.
1.2 Key Responsibilities

Authentication Hub: Discord OAuth2 token exchange and JWT issuance
API Gateway: RESTful endpoints for launcher, game server, and public access
Player Portal: Account management, statistics, leaderboards, faction info
Admin Panel: News management, player moderation, server configuration
Update Distribution: Manage client versions and patch manifests
Real-Time Status: WebSocket connections for live server status

1.3 Integration Points
SystemDirectionProtocolPurposeLauncherInboundREST APIAuth, status, news, updatesModernUO ServerBidirectionalREST + EventsPlayer sync, token validationDiscord APIOutboundOAuth2 + RESTAuthentication, webhooksCDN (Cloudflare)OutboundHTTPSUpdate file hostingPostgreSQLInternalTCPPrimary data storeRedisInternalTCPCaching, sessions, real-time
1.4 Technology Stack Summary
LayerTechnologyVersionRuntime.NET 1010.0Web FrameworkASP.NET Core10.0ORMEntity Framework Core10.0DatabasePostgreSQL16CacheRedis7.xFrontendBlazor Server / React8.0 / 18Real-TimeSignalR10.0AuthJWT + Discord OAuth2-

2. System Architecture Overview
2.1 High-Level Architecture
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL CLIENTS                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                    │
│    │   Launcher   │    │   Browser    │    │  ModernUO    │                    │
│    │   (WPF)      │    │   (Users)    │    │  Server      │                    │
│    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘                    │
└───────────┼───────────────────┼───────────────────┼────────────────────────────┘
            │ REST API          │ HTTPS             │ REST + WebSocket
┌───────────┴───────────────────┴───────────────────┴────────────────────────────┐
│                           51alpha WEB APPLICATION                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                          PRESENTATION LAYER                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │  │
│  │  │ API         │  │ Public      │  │ Admin       │  │ SignalR     │      │  │
│  │  │ Controllers │  │ Website     │  │ Panel       │  │ Hubs        │      │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘      │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                          APPLICATION LAYER                                │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │  │
│  │  │ AuthService │  │ PlayerSvc   │  │ NewsSvc     │  │ UpdateSvc   │      │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │  │
│  │  │ FactionSvc  │  │ AdminSvc    │  │ GameSyncSvc │  │ WebhookSvc  │      │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘      │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                          INFRASTRUCTURE LAYER                             │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │  │
│  │  │ EF Core     │  │ Redis       │  │ HttpClient  │  │ Background  │      │  │
│  │  │ DbContext   │  │ Cache       │  │ Factory     │  │ Services    │      │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘      │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
            │                                       │
            ▼                                       ▼
    ┌───────────────┐                      ┌───────────────┐
    │  PostgreSQL   │                      │    Redis      │
    └───────────────┘                      └───────────────┘
2.2 Project Structure
51alpha.Website/
├── 51alpha.Website.sln
├── src/
│   ├── 51alpha.Api/                      # ASP.NET Core Web API
│   │   ├── Controllers/
│   │   │   ├── AuthController.cs
│   │   │   ├── StatusController.cs
│   │   │   ├── NewsController.cs
│   │   │   ├── LauncherController.cs
│   │   │   ├── PlayersController.cs
│   │   │   ├── FactionsController.cs
│   │   │   └── AdminController.cs
│   │   ├── Hubs/
│   │   │   └── StatusHub.cs
│   │   ├── Middleware/
│   │   ├── Filters/
│   │   └── Program.cs
│   │
│   ├── 51alpha.Application/              # Business Logic
│   │   ├── Services/
│   │   ├── DTOs/
│   │   └── Validators/
│   │
│   ├── 51alpha.Domain/                   # Domain Models
│   │   ├── Entities/
│   │   ├── Enums/
│   │   └── Events/
│   │
│   ├── 51alpha.Infrastructure/           # Data Access & External
│   │   ├── Data/
│   │   ├── Repositories/
│   │   ├── External/
│   │   └── Caching/
│   │
│   ├── 51alpha.Web/                      # Public Website
│   │   ├── Pages/
│   │   ├── Components/
│   │   └── wwwroot/
│   │
│   └── 51alpha.Admin/                    # Admin Panel
│       ├── Pages/
│       ├── Components/
│       └── wwwroot/
│
├── tests/
│   ├── 51alpha.Api.Tests/
│   ├── 51alpha.Application.Tests/
│   └── 51alpha.Integration.Tests/
│
└── deploy/
    ├── docker-compose.yml
    ├── Dockerfile
    └── nginx.conf

3. Technology Stack
3.1 Backend Technologies
CategoryTechnologyPurposeRationaleRuntime.NET 10 LTSApplication runtimeMatches ModernUO, long-term supportWeb FrameworkASP.NET Core 10HTTP handling, routingHigh performance, native .NETORMEF Core 10Database accessCode-first, migrations, LINQValidationFluentValidationRequest validationExpressive, testable rulesMappingMapsterDTO mappingFast, minimal configAuthJWT BearerToken authenticationStateless, scalableReal-TimeSignalRWebSocket abstractionNative .NET, fallback supportBackground JobsHangfireScheduled tasksDashboard, persistenceLoggingSerilogStructured loggingConsistent with Launcher/Server
3.2 Data Storage
StoreTechnologyPurposePrimary DBPostgreSQL 16User accounts, players, news, audit logsCacheRedis 7Session cache, rate limiting, real-time stateFile StorageCloudflare R2 / S3Update files, player avatarsSearch (optional)MeilisearchPlayer/guild search
3.3 NuGet Packages
xml<!-- Core -->
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="10.0.0" />
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.0.0" />
<PackageReference Include="Microsoft.AspNetCore.SignalR" Version="10.0.0" />

<!-- Application -->
<PackageReference Include="FluentValidation.AspNetCore" Version="11.3.0" />
<PackageReference Include="Mapster" Version="7.4.0" />
<PackageReference Include="Hangfire.AspNetCore" Version="1.8.6" />
<PackageReference Include="Hangfire.PostgreSql" Version="1.20.0" />

<!-- Infrastructure -->
<PackageReference Include="StackExchange.Redis" Version="2.7.10" />
<PackageReference Include="Serilog.AspNetCore" Version="10.0.0" />
<PackageReference Include="Polly.Extensions.Http" Version="3.0.0" />

<!-- Security -->
<PackageReference Include="AspNetCoreRateLimit" Version="5.0.0" />
<PackageReference Include="BCrypt.Net-Next" Version="4.0.3" />

4. Database Schema
4.1 Entity Relationship Diagram
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│      Users       │       │     Players      │       │   Characters     │
├──────────────────┤       ├──────────────────┤       ├──────────────────┤
│ Id (PK)          │──┐    │ Id (PK)          │──┐    │ Id (PK)          │
│ DiscordId (UK)   │  │    │ UserId (FK)      │◀─┘    │ PlayerId (FK)    │◀─┐
│ Username         │  │    │ AccountName      │       │ Name             │  │
│ Email            │  └───▶│ CreatedAt        │       │ FactionId (FK)   │──┼──┐
│ AvatarUrl        │       │ LastLoginAt      │       │ FactionRank      │  │  │
│ Roles            │       │ IsBanned         │       │ TotalKills       │  │  │
│ CreatedAt        │       │ BanReason        │       │ TotalDeaths      │  │  │
│ LastLoginAt      │       │ BanExpiresAt     │       │ FactionPoints    │  │  │
└──────────────────┘       └──────────────────┘       │ TalismanXP       │  │  │
                                                      │ CreatedAt        │  │  │
┌──────────────────┐       ┌──────────────────┐       └──────────────────┘  │  │
│    Factions      │       │  FactionTowns    │                             │  │
├──────────────────┤       ├──────────────────┤                             │  │
│ Id (PK)          │◀──────│ FactionId (FK)   │◀────────────────────────────┘  │
│ Name             │  ┌───▶│ TownName         │                                │
│ Description      │  │    │ ControlledSince  │                                │
│ Color            │  │    │ TaxRate          │                                │
│ TotalMembers     │──┘    └──────────────────┘                                │
│ TotalPoints      │                                                           │
│ TownsControlled  │                                                           │
└──────────────────┘                                                           │
4.2 Entity Definitions
4.2.1 User Entity
csharpnamespace _51alpha.Domain.Entities;

public class User
{
    public Guid Id { get; set; }
    public string DiscordId { get; set; } = string.Empty;
    public string Username { get; set; } = string.Empty;
    public string? Email { get; set; }
    public string? AvatarUrl { get; set; }
    public UserRole[] Roles { get; set; } = Array.Empty<UserRole>();
    public DateTime CreatedAt { get; set; }
    public DateTime LastLoginAt { get; set; }
    
    // Navigation
    public Player? Player { get; set; }
    public ICollection<RefreshToken> RefreshTokens { get; set; } = new List<RefreshToken>();
}

public enum UserRole
{
    Player = 0,
    Moderator = 1,
    Admin = 2,
    SuperAdmin = 3
}
4.2.2 Player Entity
csharppublic class Player
{
    public Guid Id { get; set; }
    public Guid UserId { get; set; }
    public string AccountName { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? LastLoginAt { get; set; }
    
    // Moderation
    public bool IsBanned { get; set; }
    public string? BanReason { get; set; }
    public DateTime? BanExpiresAt { get; set; }
    
    // Navigation
    public User User { get; set; } = null!;
    public ICollection<Character> Characters { get; set; } = new List<Character>();
}
4.2.3 Character Entity
csharppublic class Character
{
    public Guid Id { get; set; }
    public Guid PlayerId { get; set; }
    public string Name { get; set; } = string.Empty;
    
    // Faction
    public Guid? FactionId { get; set; }
    public int FactionRank { get; set; }
    public int FactionPoints { get; set; }
    
    // Stats (synced from game server)
    public int TotalKills { get; set; }
    public int TotalDeaths { get; set; }
    public int TalismanXP { get; set; }
    public int TalismanLevel { get; set; }
    
    // Navigation
    public Player Player { get; set; } = null!;
    public Faction? Faction { get; set; }
}
4.2.4 NewsArticle Entity
csharppublic class NewsArticle
{
    public Guid Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Summary { get; set; } = string.Empty;
    public string? Content { get; set; }  // Markdown
    public string? ImageUrl { get; set; }
    public Guid AuthorId { get; set; }
    public NewsCategory Category { get; set; }
    public bool IsPinned { get; set; }
    public bool IsPublished { get; set; }
    public DateTime? PublishedAt { get; set; }
    
    public User Author { get; set; } = null!;
}

public enum NewsCategory
{
    Announcement,
    PatchNotes,
    Event,
    Maintenance,
    Community
}
4.2.5 LauncherVersion Entity
csharppublic class LauncherVersion
{
    public Guid Id { get; set; }
    public string Version { get; set; } = string.Empty;  // SemVer
    public string MinimumVersion { get; set; } = string.Empty;
    public string DownloadUrl { get; set; } = string.Empty;
    public string Checksum { get; set; } = string.Empty;  // SHA256
    public long FileSizeBytes { get; set; }
    public string? ReleaseNotes { get; set; }
    public bool IsForced { get; set; }
    public bool IsCurrent { get; set; }
    
    public ICollection<PatchFile> PatchFiles { get; set; } = new List<PatchFile>();
}

5. Authentication System
5.1 OAuth2 Flow Overview
     Launcher                 51alpha API              Discord API
        │                         │                        │
        │  1. POST /auth/discord  │                        │
        │     { code, redirect }  │                        │
        │────────────────────────▶│                        │
        │                         │  2. POST /oauth2/token │
        │                         │───────────────────────▶│
        │                         │                        │
        │                         │  3. { access_token }   │
        │                         │◀───────────────────────│
        │                         │                        │
        │                         │  4. GET /users/@me     │
        │                         │───────────────────────▶│
        │                         │                        │
        │                         │  5. { id, username }   │
        │                         │◀───────────────────────│
        │                         │                        │
        │  7. { jwt, refresh,     │  6. Create user,       │
        │       user, expires }   │     generate JWT       │
        │◀────────────────────────│                        │
5.2 JWT Configuration
csharppublic class JwtSettings
{
    public string SecretKey { get; set; } = string.Empty;
    public string Issuer { get; set; } = "51alpha";
    public string Audience { get; set; } = "51alpha-clients";
    public int AccessTokenExpirationMinutes { get; set; } = 60;
    public int RefreshTokenExpirationDays { get; set; } = 30;
}
5.3 AuthService Implementation
csharppublic class AuthService : IAuthService
{
    public async Task<AuthResult> AuthenticateWithDiscordAsync(
        string code, string redirectUri, CancellationToken ct = default)
    {
        // 1. Exchange code for Discord tokens
        var discordTokens = await _discord.ExchangeCodeAsync(code, redirectUri, ct);
        
        // 2. Get Discord user info
        var discordUser = await _discord.GetCurrentUserAsync(discordTokens.AccessToken, ct);
        
        // 3. Find or create user
        var user = await _db.Users.FirstOrDefaultAsync(u => u.DiscordId == discordUser.Id, ct)
            ?? new User { DiscordId = discordUser.Id, Username = discordUser.Username };
        
        // 4. Generate tokens
        var accessToken = GenerateAccessToken(user);
        var refreshToken = GenerateRefreshToken();
        
        // 5. Store refresh token and return
        return AuthResult.Success(new AuthResponse
        {
            AccessToken = accessToken,
            RefreshToken = refreshToken,
            ExpiresIn = _jwtSettings.AccessTokenExpirationMinutes * 60,
            User = MapToUserInfo(user)
        });
    }
}

6. API Endpoints
6.1 Endpoint Summary
MethodEndpointAuthDescriptionPOST/api/auth/discordNoExchange Discord code for JWTPOST/api/auth/refreshNoRefresh access tokenPOST/api/auth/logoutYesRevoke refresh tokenGET/api/statusNoServer statusGET/api/newsNoList news articlesGET/api/news/{id}NoGet single articleGET/api/launcher/versionNoCurrent launcher versionGET/api/launcher/patchesNoDifferential patchesGET/api/meYesCurrent user infoGET/api/players/{id}NoPlayer profileGET/api/factionsNoAll factionsGET/api/leaderboards/{type}NoLeaderboard data
6.2 Authentication Endpoints
POST /api/auth/discord
json// Request
{
  "code": "discord_authorization_code",
  "redirect_uri": "http://localhost:51401/callback"
}

// Response 200 OK
{
  "success": true,
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "dGhpcyBpcyBhIHJlZnJl...",
  "expires_in": 3600,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "discord_id": "123456789012345678",
    "username": "PlayerOne",
    "roles": ["Player"],
    "player": {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "account_name": "PlayerOne",
      "faction": "True Britannians"
    }
  }
}
6.3 Status Endpoint
GET /api/status
json{
  "online": true,
  "players": 127,
  "max_players": 500,
  "uptime_hours": 72.5,
  "next_siege_window": "2024-12-15T20:00:00Z",
  "next_event": "2024-12-14T18:00:00Z",
  "event_name": "Double XP Weekend",
  "health": "healthy",
  "server_version": "1.0.0"
}
6.4 Launcher Endpoints
GET /api/launcher/version
json{
  "current_version": "1.2.5",
  "minimum_version": "1.2.0",
  "update_url": "https://cdn.51alpha.com/launcher/51alpha-launcher-1.2.5.zip",
  "checksum": "A1B2C3D4E5F6...",
  "file_size_bytes": 45678901,
  "is_forced": false,
  "release_notes": "## What's New\n- Fixed faction point bug"
}
GET /api/launcher/patches?from=1.2.2
json{
  "from_version": "1.2.2",
  "to_version": "1.2.5",
  "patches": [
    {
      "file_name": "ClassicUO.dll",
      "relative_path": "ClassicUO.dll",
      "url": "https://cdn.51alpha.com/patches/1.2.5/ClassicUO.dll",
      "checksum": "F1E2D3C4B5A6...",
      "size_bytes": 2345678,
      "action": "update"
    }
  ],
  "total_size_bytes": 5678901
}

7. Admin Panel
7.1 Admin Features
FeatureDescriptionRequired RoleDashboardServer stats, player activity graphsModerator+News ManagementCreate/edit/delete news articlesModerator+Player LookupSearch players, view detailsModerator+Ban ManagementIssue/lift bans, view historyModerator+Faction AdminAdjust faction points, town controlAdmin+Version ManagementUpload new versions, manage patchesAdmin+User ManagementAssign roles, manage adminsSuperAdminAudit LogsView all admin actionsAdmin+
7.2 Admin API Endpoints
// All admin endpoints require [Authorize(Roles = "Moderator,Admin,SuperAdmin")]

// News Management
POST   /api/admin/news                    // Create article
PUT    /api/admin/news/{id}               // Update article
DELETE /api/admin/news/{id}               // Delete article

// Player Management
GET    /api/admin/players                 // Search players
POST   /api/admin/players/{id}/ban        // Issue ban
POST   /api/admin/players/{id}/unban      // Lift ban

// Version Management (Admin+ only)
POST   /api/admin/versions                // Create new version
POST   /api/admin/versions/{id}/activate  // Set as current

8. Public Website
8.1 Page Structure
PageRouteDescriptionHome/Landing page, server status, featured newsNews/newsNews listing with categoriesLeaderboards/leaderboardsKill/faction rankingsFactions/factionsFaction overviewPlayer Profile/players/{name}Public player statsDownload/downloadLauncher downloadAbout/aboutServer info, rules

9. Game Server Integration
9.1 Integration Architecture
┌──────────────────┐                              ┌──────────────────┐
│   51alpha API    │                              │  ModernUO Server │
├──────────────────┤                              ├──────────────────┤
│                  │     Token Validation         │                  │
│  POST /api/game/ │◀─────────────────────────────│  Player Login    │
│  validate-token  │                              │                  │
│                  │─────────────────────────────▶│  { valid, user } │
│                  │                              │                  │
│                  │     Player Stats Sync        │                  │
│  POST /api/game/ │◀─────────────────────────────│  On Kill/Death   │
│  sync-stats      │                              │  Periodic Batch  │
│                  │                              │                  │
│                  │     Server Status            │                  │
│  POST /api/game/ │◀─────────────────────────────│  Heartbeat       │
│  heartbeat       │                              │  (every 30s)     │
└──────────────────┘                              └──────────────────┘
9.2 Game Server API Endpoints
// All game server endpoints require API key authentication
// Header: X-API-Key: {game_server_api_key}

// Token Validation
POST /api/game/validate-token
Request:  { "token": "jwt_access_token" }
Response: { "valid": true, "user_id": "...", "player_id": "...", "account_name": "..." }

// Stats Sync (batch)
POST /api/game/sync-stats
Request: {
  "characters": [
    {
      "character_name": "Lord Blackthorn",
      "kills": 5,
      "deaths": 2,
      "faction_points": 150,
      "talisman_xp": 2500
    }
  ]
}

// Server Heartbeat
POST /api/game/heartbeat
Request: {
  "players_online": 127,
  "uptime_seconds": 259200,
  "memory_mb": 2048
}

10. CDN & File Management
10.1 File Storage Strategy
File TypeStorageCDNCache TTLLauncher UpdatesCloudflare R2Cloudflare1 year (versioned)Patch FilesCloudflare R2Cloudflare1 year (versioned)News ImagesCloudflare R2Cloudflare30 daysUser AvatarsDiscord CDNDiscordN/A

11. Real-Time Features
11.1 SignalR Hub
csharppublic interface IStatusHubClient
{
    Task ServerStatus(ServerStatusDto status);
    Task PlayerCountUpdate(int count);
    Task FactionEvent(FactionEventDto factionEvent);
    Task NewsPublished(NewsItemDto news);
}

[Authorize]
public class StatusHub : Hub<IStatusHubClient>
{
    public override async Task OnConnectedAsync()
    {
        // Send current status immediately
        var status = await _cache.GetStringAsync("server:status");
        await Clients.Caller.ServerStatus(status);
        await base.OnConnectedAsync();
    }
    
    public async Task SubscribeToFaction(string factionName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"faction:{factionName}");
    }
}

12. Security Implementation
12.1 Security Checklist

HTTPS everywhere with TLS 1.2+ minimum
JWT tokens with short expiration (1 hour)
Refresh token rotation on each use
API key authentication for game server
Rate limiting on all endpoints
CORS restricted to known origins
SQL injection prevention via EF Core
Input validation with FluentValidation
Audit logging for admin actions
Secrets in environment variables

12.2 Rate Limiting
csharpbuilder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", config =>
    {
        config.Window = TimeSpan.FromMinutes(1);
        config.PermitLimit = 60;
    });
    
    options.AddFixedWindowLimiter("auth", config =>
    {
        config.Window = TimeSpan.FromMinutes(15);
        config.PermitLimit = 10;
    });
});

13. Deployment & Infrastructure
13.1 Docker Compose
yamlversion: '3.8'

services:
  api:
    build: .
    ports:
      - "5000:80"
    environment:
      - ConnectionStrings__Database=Host=postgres;Database=51alpha;...
      - ConnectionStrings__Redis=redis:6379
      - Jwt__SecretKey=${JWT_SECRET}
      - Discord__ClientId=${DISCORD_CLIENT_ID}
      - Discord__ClientSecret=${DISCORD_CLIENT_SECRET}
    depends_on:
      - postgres
      - redis
    
  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
13.2 Environment Variables
VariableDescriptionRequiredConnectionStrings__DatabasePostgreSQL connection stringYesConnectionStrings__RedisRedis connection stringYesJwt__SecretKeyJWT signing key (min 32 chars)YesDiscord__ClientIdDiscord OAuth client IDYesDiscord__ClientSecretDiscord OAuth client secretYesGameServer__ApiKeyAPI key for game serverYes

14. Monitoring & Observability
14.1 Health Checks
csharpbuilder.Services.AddHealthChecks()
    .AddNpgSql(connectionString, name: "database")
    .AddRedis(redisConnection, name: "redis")
    .AddCheck<GameServerHealthCheck>("game_server");

app.MapHealthChecks("/health");

15. Testing Strategy
15.1 Test Coverage Targets
LayerTargetPriorityApplication Services90%CriticalAPI Controllers85%CriticalDomain Entities80%HighInfrastructure70%High

16. Implementation Phases
Phase 1: Foundation (Week 1)

Create solution structure
Set up EF Core with PostgreSQL
Create initial migrations
Configure Serilog logging
Set up Docker Compose

Acceptance: Database migrations run, API starts, health check passes
Phase 2: Authentication (Week 2)

Implement Discord OAuth client
Create AuthService with token exchange
Implement JWT generation and validation
Create refresh token rotation
Build AuthController endpoints

Acceptance: Launcher can authenticate via Discord and receive JWT
Phase 3: Core API (Week 2-3)

Implement StatusController with Redis caching
Create NewsService and NewsController
Build LauncherController (version, patches)
Implement rate limiting

Acceptance: All launcher-required endpoints functional
Phase 4: Game Integration (Week 3)

Create GameController with API key auth
Implement token validation endpoint
Build stats sync service
Create faction event processing

Acceptance: Game server can validate tokens and sync data
Phase 5: Admin Panel (Week 4)

Create Blazor Server admin project
Build dashboard with charts
Implement news management UI
Create player management

Acceptance: Admins can manage news, players, and versions
Phase 6: Public Website (Week 4-5)

Create public website
Build homepage with status and news
Implement leaderboards
Create faction pages

Acceptance: Public website live with all pages functional
Phase 7: Real-Time & Polish (Week 5)

Implement SignalR hub
Add real-time status updates
Create audit logging
Set up CI/CD pipeline

Acceptance: Full system deployed and operational

17. Appendices
A. API Error Codes
CodeHTTP StatusDescriptioninvalid_code400Discord authorization code invalidtoken_expired401Refresh token has expiredaccount_banned403Player account is bannedrate_limit_exceeded429Too many requestsnot_found404Resource not found
B. Configuration Schema
json{
  "ConnectionStrings": {
    "Database": "Host=localhost;Database=51alpha;Username=app;Password=xxx",
    "Redis": "localhost:6379"
  },
  "Jwt": {
    "SecretKey": "your-secret-key-min-32-characters",
    "Issuer": "51alpha",
    "Audience": "51alpha-clients",
    "AccessTokenExpirationMinutes": 60,
    "RefreshTokenExpirationDays": 30
  },
  "Discord": {
    "ClientId": "your-client-id",
    "ClientSecret": "your-client-secret"
  },
  "GameServer": {
    "ApiKey": "your-game-server-api-key"
  }
}
C. Glossary
TermDefinitionJWTJSON Web Token for stateless authenticationOAuth2Authorization framework for third-party loginSignalRLibrary for real-time web functionalityEF CoreEntity Framework Core ORMR2Cloudflare's S3-compatible object storage

End of Document
51alpha Website Architecture Specification v1.0
Web API • Player Portal • Admin Panel
Prepared for AI-Assisted Development