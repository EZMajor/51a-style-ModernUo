51alpha Integration Map
Version 1.0 | December 2024
Launcher ↔ Website API ↔ Game Server
Complete System Integration Reference

Table of Contents

System Overview
Component Summary
Network Topology
Authentication Flow
API Contract Matrix
Data Synchronization
Real-Time Communication
Sequence Diagrams
Shared Data Models
Configuration Alignment
Error Handling Across Systems
Deployment Topology
Development Workflow
Testing Integration Points
Monitoring & Observability
Troubleshooting Guide


1. System Overview
1.1 The 51alpha Ecosystem
The 51alpha project consists of three primary systems that work together to deliver a complete gaming experience. This document defines exactly how these systems communicate, share data, and maintain consistency.
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              51alpha ECOSYSTEM                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

    ┌─────────────────────┐                                    ┌─────────────────────┐
    │   LAUNCHER (WPF)    │                                    │   CLASSICUO CLIENT  │
    │   • Discord OAuth   │                                    │   • Game Rendering  │
    │   • Auto-Updates    │                                    │   • Player Input    │
    │   • Server Status   │                                    │   • UO Protocol     │
    └─────────┬───────────┘                                    └──────────┬──────────┘
              │ HTTPS/REST                                                │ TCP/UO
              ▼                                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              WEBSITE / API (ASP.NET Core)                           │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                    │
│   │   REST API      │  │  Admin Panel    │  │  Public Site    │                    │
│   └────────┬────────┘  └─────────────────┘  └─────────────────┘                    │
│            │                                                                        │
│   ┌────────┴────────┐  ┌─────────────────┐  ┌─────────────────┐                   │
│   │   PostgreSQL    │  │     Redis       │  │   Cloudflare    │                   │
│   └──────────────────┘  └─────────────────┘  └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
              │ HTTPS/REST (API Key)                          ▲
              ▼                                               │
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              GAME SERVER (ModernUO)                                 │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                    │
│   │   UO Protocol   │  │  Game Logic     │  │  Web API Client │                    │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘                    │
│   FactionEngine • TalismanSystem • SiegeManager • TournamentManager • BODSystem    │
└─────────────────────────────────────────────────────────────────────────────────────┘
1.2 Communication Channels
ChannelFromToProtocolAuth MethodLauncher → APILauncherWebsiteHTTPS RESTJWT BearerAPI → DiscordWebsiteDiscordHTTPS RESTOAuth2Game → APIModernUOWebsiteHTTPS RESTAPI KeyAPI → GameWebsiteModernUOHTTPS RESTAPI KeyClient → GameClassicUOModernUOTCPUO ProtocolBrowser → APIUserWebsiteHTTPSJWT/SessionSignalRAPILauncher/BrowserWebSocketJWT

2. Component Summary
2.1 Launcher (WPF/.NET 10)
AspectDetailsRepositorygithub.com/EZMajor/51a-style-ModernUo (launcher branch)TechnologyWPF, .NET 10, C# 13Primary RolePlayer entry point, authentication, updatesConsumesWebsite REST APIProducesOAuth callbacks, user sessionsKey ServicesAuthService, UpdateService, LaunchService
2.2 Website/API (ASP.NET Core 10)
AspectDetailsRepositorygithub.com/EZMajor/51a-style-ModernUo (website branch)TechnologyASP.NET Core 10, EF Core, PostgreSQL, RedisPrimary RoleCentral hub, API gateway, admin panelConsumesDiscord API, Game Server APIProducesREST API, SignalR, Public WebsiteKey ServicesAuthService, GameSyncService, UpdateService
2.3 Game Server (ModernUO)
AspectDetailsRepositorygithub.com/EZMajor/51a-style-ModernUo (main branch)TechnologyModernUO, .NET 10, C# 13Primary RoleGame world, player sessions, game logicConsumesWebsite API (token validation)ProducesPlayer stats, faction events, heartbeatsKey SystemsFactionEngine, TalismanSystem, SiegeManager

3. Network Topology
3.1 Domain Configuration
DomainPoints ToPurpose51alpha.comWebsite (public)Landing page, news, leaderboardsapi.51alpha.comWebsite (API)REST API for launcher/gameadmin.51alpha.comWebsite (admin)Admin panelcdn.51alpha.comCloudflare R2Update files, assetsplay.51alpha.comGame ServerUO client connection
3.2 Port Configuration
PortProtocolServiceExposed To443HTTPSWebsite APIInternet2593TCPGame Server (UO)Internet5000HTTPWebsite (internal)Reverse proxy only5432TCPPostgreSQLInternal only6379TCPRedisInternal only

4. Authentication Flow
4.1 Complete Auth Journey
  PHASE 1: DISCORD OAUTH (Launcher → Discord → API)
  ═══════════════════════════════════════════════════

  Launcher              Browser              Discord              51alpha API
     │                     │                    │                      │
     │ 1. Click Login      │                    │                      │
     ├────────────────────▶│                    │                      │
     │                     │ 2. Open Auth URL   │                      │
     │                     ├───────────────────▶│                      │
     │                     │                    │ 3. User Authorizes   │
     │                     │ 4. Redirect with   │                      │
     │◀────────────────────┤    code + state    │                      │
     │ 5. Receive callback │                    │                      │
     │                     │                    │                      │
     │ 6. POST /api/auth/discord               │                      │
     ├─────────────────────────────────────────────────────────────────▶
     │                                                                 │
     │ 9. { jwt, refresh_token, user }                                 │
     │◀────────────────────────────────────────────────────────────────┤


  PHASE 2: GAME LOGIN (Launcher → Game Server → API)
  ═══════════════════════════════════════════════════

  Launcher              ClassicUO            Game Server          51alpha API
     │                     │                    │                      │
     │ 11. Launch game     │                    │                      │
     │     with JWT        │                    │                      │
     ├────────────────────▶│                    │                      │
     │                     │ 12. Connect        │                      │
     │                     ├───────────────────▶│                      │
     │                     │                    │ 13. POST /api/game/  │
     │                     │                    │     validate-token   │
     │                     │                    ├─────────────────────▶│
     │                     │                    │ 14. { valid, user }  │
     │                     │                    │◀─────────────────────┤
     │                     │ 15. Login success  │                      │
     │                     │◀───────────────────┤                      │
4.2 Token Types & Lifetimes
Token TypeIssued ByLifetimeStoragePurposeDiscord AccessDiscord7 daysNot storedExchange for user info51alpha JWTWebsite API1 hourLauncher (DPAPI)API authenticationRefresh TokenWebsite API30 daysLauncher (DPAPI) + DBGet new JWTGame API KeyManualPermanentServer configGame→API auth
4.3 JWT Claims
json{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "name": "PlayerOne",
  "discord_id": "123456789012345678",
  "player_id": "660e8400-e29b-41d4-a716-446655440001",
  "account_name": "PlayerOne",
  "role": ["Player"],
  "iss": "51alpha",
  "aud": "51alpha-clients",
  "exp": 1702234567,
  "iat": 1702230967
}

5. API Contract Matrix
5.1 Launcher → Website API
EndpointMethodAuthRequestResponse/api/auth/discordPOSTNone{ code, redirect_uri }{ jwt, refresh, user }/api/auth/refreshPOSTNone{ refresh_token }{ jwt, refresh }/api/auth/logoutPOSTJWT{ refresh_token }204 No Content/api/statusGETNone-{ online, players, uptime }/api/newsGETNone?page&limit{ items[], total }/api/launcher/versionGETNone-{ version, url, checksum }/api/launcher/patchesGETNone?from=version{ patches[] }/api/meGETJWT-{ user, player, characters }
5.2 Game Server → Website API
EndpointMethodAuthRequestResponse/api/game/validate-tokenPOSTAPI Key{ token }{ valid, user_id, player_id }/api/game/sync-statsPOSTAPI Key{ characters[] }{ processed, errors }/api/game/faction-eventPOSTAPI Key{ event_type, faction, details }{ acknowledged }/api/game/heartbeatPOSTAPI Key{ players, uptime, memory }{ acknowledged }
5.3 SignalR Hub Methods
HubMethodDirectionPayloadStatusHubServerStatusServer→Client{ online, players, uptime }StatusHubPlayerCountUpdateServer→Clientint countStatusHubFactionEventServer→Client{ event_type, faction, details }StatusHubNewsPublishedServer→Client{ id, title, summary }StatusHubSubscribeToFactionClient→Serverstring factionName

6. Data Synchronization
6.1 Data Ownership
  DATA ENTITY                 OWNER              CONSUMERS              SYNC METHOD
  ─────────────────────────────────────────────────────────────────────────────────────
  
  User Account                Website            Launcher, Game         JWT claims
  Player Record               Website            Game                   Token validation
  
  Character Stats             Game Server        Website                POST /sync-stats
  ├── Kills/Deaths            Game (source)      Website (mirror)       Batch every 5 min
  ├── Faction Points          Game (source)      Website (mirror)       Real-time on change
  └── Talisman XP             Game (source)      Website (mirror)       Batch every 5 min
  
  Faction State               Game Server        Website                POST /faction-event
  
  Server Status               Game Server        Website, Launcher      POST /heartbeat
  
  News Articles               Website            Launcher               GET /news
  Launcher Versions           Website            Launcher               GET /version
6.2 Sync Timing
Data TypeTriggerFrequencyMethodServer StatusTimerEvery 30 secondsGame → POST /heartbeatCharacter StatsTimer + Events5 min batch + instant on PvP killGame → POST /sync-statsFaction EventsEvent-drivenInstantGame → POST /faction-eventPlayer LoginPlayer connectsInstantGame → POST /player-loginLeaderboardsScheduledEvery 15 minutesWebsite background job
6.3 Conflict Resolution

Character Stats: Game Server is always authoritative - Website mirrors
User Data: Website is always authoritative - Game reads via token
Ban Status: Website is authoritative - Game checks on each login
Faction Points: Game calculates, Website aggregates for display


7. Real-Time Communication
7.1 SignalR Architecture
  Game Server                    Website API                    Clients
       │                              │                            │
       │  POST /api/game/heartbeat    │                            │
       ├─────────────────────────────▶│                            │
       │  { players: 127 }            │                            │
       │                              │  Update Redis cache        │
       │                              │  SignalR: ServerStatus     │
       │                              ├───────────────────────────▶│
       │                              │  { online: true,           │
       │                              │    players: 127 }          │
7.2 WebSocket Connection
csharp// Launcher Connection
var connection = new HubConnectionBuilder()
    .WithUrl("https://api.51alpha.com/hubs/status", options =>
    {
        options.AccessTokenProvider = () => Task.FromResult(_authService.GetAccessToken());
    })
    .WithAutomaticReconnect()
    .Build();

connection.On<ServerStatus>("ServerStatus", status => {
    // Update UI with new status
});

await connection.StartAsync();

8. Sequence Diagrams
8.1 Player Login (Complete Flow)
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│Launcher│  │Browser │  │Discord │  │ API    │  │  DB    │  │ Game   │
└───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘
    │ Click Login           │           │           │           │
    ├──────────▶│ Open Auth │           │           │           │
    │           ├──────────▶│ Authorize │           │           │
    │           │◀──────────┤ code      │           │           │
    │◀──────────┤ callback  │           │           │           │
    │ POST /auth/discord    │           │           │           │
    ├───────────────────────────────────▶ Exchange  │           │
    │           │           │◀──────────┤ tokens    │           │
    │           │           ├──────────▶│ Upsert    │           │
    │           │           │           ├──────────▶│           │
    │◀──────────────────────────────────┤ JWT+user  │           │
    │ Launch ClassicUO with JWT         │           │           │
    ├───────────────────────────────────────────────────────────▶
    │           │           │           │◀──────────────────────┤
    │           │           │           │ validate-token        │
    │           │           │           ├──────────────────────▶│
    │           │           │           │           │  valid    │
    │◀──────────────────────────────────────────────────────────┤
    │           │           │           │           │  Playing! │

9. Shared Data Models
9.1 Cross-System DTOs
ServerStatus
json{
  "online": true,
  "players": 127,
  "max_players": 500,
  "uptime_hours": 72.5,
  "next_siege_window": "2024-12-15T20:00:00Z",
  "health": "healthy",
  "server_version": "1.0.0"
}
CharacterStatUpdate
json{
  "character_name": "Lord Blackthorn",
  "kills": 5,
  "deaths": 2,
  "faction_points": 150,
  "talisman_xp": 2500
}
FactionEvent
json{
  "event_type": "town_captured",
  "faction": "True Britannians",
  "town": "Britain",
  "details": {
    "previous_owner": "Shadowlords",
    "siege_duration_minutes": 45
  }
}
TokenValidationResult
json{
  "valid": true,
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "player_id": "660e8400-e29b-41d4-a716-446655440001",
  "account_name": "PlayerOne",
  "is_banned": false
}
9.2 Enum Alignment
csharp// Must match across all systems
enum NewsCategory { Announcement, PatchNotes, Event, Maintenance, Community }
enum UserRole { Player = 0, Moderator = 1, Admin = 2, SuperAdmin = 3 }
enum FactionName { TrueBritannians, CouncilOfMages, Shadowlords, Minax }
enum ServerHealth { Healthy, Degraded, Maintenance, Offline }

10. Configuration Alignment
10.1 Shared Configuration Values
SettingLauncherWebsiteGame ServerAPI Base URLhttps://api.51alpha.comN/A (self)https://api.51alpha.comCDN Base URLhttps://cdn.51alpha.comhttps://cdn.51alpha.comN/AGame Server Addressplay.51alpha.comN/AN/A (self)Game Server Port2593N/A2593JWT Issuer51alpha51alpha51alphaJWT Audience51alpha-clients51alpha-clients51alpha-clientsOAuth Callback Port51401N/AN/A
10.2 Environment Variables
bash# WEBSITE / API
ConnectionStrings__Database=Host=postgres;Database=51alpha;Username=app;Password=xxx
ConnectionStrings__Redis=redis:6379
Jwt__SecretKey=your-256-bit-secret-key-here-min-32-chars
Discord__ClientId=your-discord-client-id
Discord__ClientSecret=your-discord-client-secret
GameServer__ApiKey=shared-api-key-for-game-server

# GAME SERVER
51ALPHA_API_URL=https://api.51alpha.com
51ALPHA_API_KEY=shared-api-key-for-game-server
51ALPHA_JWT_SECRET=your-256-bit-secret-key-here-min-32-chars  # Same as Website!
10.3 Critical Alignment Requirements

JWT Secret MUST be identical between Website and Game Server
Discord Client ID MUST be identical between Website and Launcher
API Key MUST match between Website config and Game Server config
JWT Issuer and Audience MUST match for token validation to work


11. Error Handling Across Systems
11.1 Error Code Registry
CodeHTTPSourceConsumer Actioninvalid_code400APILauncher: Show 'auth failed', retrytoken_expired401APILauncher: Auto-refresh, Game: reject logintoken_invalid401APILauncher: Clear tokens, show loginaccount_banned403APILauncher: Show ban message, Game: disconnectrate_limit_exceeded429APIAll: Wait retry_after secondsserver_maintenance503APIAll: Show maintenance message
11.2 Retry Strategies
csharp// Launcher → API retry policy (Polly)
Policy
    .Handle<HttpRequestException>()
    .OrResult<HttpResponseMessage>(r => r.StatusCode >= 500)
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt))
    );

// Circuit Breaker
Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30)
    );

12. Deployment Topology
12.1 Docker Compose
yamlversion: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - api
    
  api:
    build: ./src
    ports:
      - "5000"
    environment:
      - ConnectionStrings__Database=Host=postgres;...
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
    
  game:
    build: ./game
    ports:
      - "2593:2593"

volumes:
  postgres_data:
  redis_data:

13. Development Workflow
13.1 Repository Structure
github.com/EZMajor/51a-style-ModernUo/
│
├── main branch (Game Server)
│   └── ModernUO fork with 51alpha customizations
│
├── launcher branch (WPF Launcher)
│   └── 51alpha.Launcher solution
│
├── website branch (ASP.NET Core)
│   └── 51alpha.Website solution
│
└── docs branch (Documentation)
    └── architecture/
13.2 Local Development Setup
bash# 1. Clone all branches
git clone https://github.com/EZMajor/51a-style-ModernUo.git

# 2. Set up Website (needed first for auth)
git checkout website
docker-compose up -d postgres redis
dotnet run --project src/51alpha.Api

# 3. Set up Launcher (separate terminal)
git checkout launcher
dotnet run --project src/51alpha.Launcher

# 4. Set up Game Server (separate terminal)
git checkout main
dotnet run

14. Testing Integration Points
14.1 Integration Test Matrix
Test CaseSystems InvolvedVerificationDiscord OAuthLauncher, API, DiscordUser created, JWT issuedToken RefreshLauncher, APINew JWT, rotated refresh tokenToken ValidationGame, APIValid response with user dataStats SyncGame, API, DBCharacter stats updatedFaction EventGame, API, SignalR, LauncherEvent received by clientsServer StatusGame, API, Redis, LauncherStatus displayed correctlyAuto-UpdateLauncher, API, CDNFiles downloaded, verified

15. Monitoring & Observability
15.1 Key Metrics
# Website API Metrics
api_requests_total{endpoint, method, status}
api_request_duration_seconds{endpoint}
api_active_connections

# Game Server Metrics
game_players_online
game_uptime_seconds
game_sync_queue_size
15.2 Alerting Rules
AlertConditionSeverityActionAPI DownHealth check fails 3xCriticalPage on-callGame Server OfflineNo heartbeat 2 minCriticalPage on-callHigh LatencyP99 > 500ms for 5 minWarningSlack notificationSync Queue GrowingQueue > 500 itemsWarningCheck API connection

16. Troubleshooting Guide
16.1 Common Issues
Launcher can't authenticate
Symptoms:
- "Login failed" error after Discord authorization

Diagnosis:
1. Check launcher logs: %LOCALAPPDATA%\51alpha\Launcher\logs\
2. Verify Discord redirect URI: http://localhost:51401/callback

Solutions:
- Ensure port 51401 is not in use
- Check Discord Developer Portal redirect URIs
Game Server can't validate tokens
Symptoms:
- Players can't connect ("authentication failed")

Diagnosis:
1. Check JWT_SECRET is identical on both systems
2. Check JWT hasn't expired (1 hour lifetime)
3. Verify clocks are synchronized (NTP)
Stats not syncing to website
Symptoms:
- Kills/deaths not appearing on leaderboards

Diagnosis:
1. Check game server sync queue size
2. Look for API errors in game logs
3. Verify API Key is correct
16.2 Quick Reference
ProblemFirst CheckLog LocationAuth failsDiscord redirect URIsAPI: /var/log/51alpha/Token invalidJWT secret matchGame: ./logs/Stats not syncingAPI Key correctGame: ./logs/sync.logSignalR failsCORS configurationBrowser: F12 consoleUpdates failCDN accessibilityLauncher: %LOCALAPPDATA%

End of Document
51alpha Integration Map v1.0
Launcher ↔ Website API ↔ Game Server
Complete System Integration Reference
