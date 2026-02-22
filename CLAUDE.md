# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Jellyfin.Xtream is a Jellyfin plugin that integrates content from Xtream-compatible IPTV APIs. It provides Live TV, Video-on-Demand (VOD), TV Series, and catch-up TV functionality. The plugin is built on .NET 9.0 and targets Jellyfin 10.11.0.

## Build Commands

```bash
# Build the plugin
dotnet build Jellyfin.Xtream.sln

# Build in release mode
dotnet build Jellyfin.Xtream.sln -c Release

# Clean build artifacts
dotnet clean Jellyfin.Xtream.sln

# Restore NuGet packages
dotnet restore Jellyfin.Xtream.sln
```

The build produces `Jellyfin.Xtream.dll` in `Jellyfin.Xtream/bin/Debug/net9.0/` (or `Release`).

## Code Quality

The project uses strict code analysis:
- **TreatWarningsAsErrors**: Enabled - all warnings must be fixed
- **StyleCop.Analyzers**: Enforces coding style (see `jellyfin.ruleset` for disabled rules)
- **SerilogAnalyzer**: Validates logging patterns
- **MultithreadingAnalyzer**: Catches concurrency issues
- **Nullable reference types**: Enabled throughout the codebase

Run build to check for code analysis violations.

## Architecture Overview

### Plugin Integration Points

The plugin implements several Jellyfin interfaces:
- **ILiveTvService** (`LiveTvService.cs`) - Provides Live TV channels and EPG data
- **IChannel** - Three implementations for content types:
  - `VodChannel` - Video-on-Demand movies
  - `SeriesChannel` - TV series with seasons/episodes
  - `CatchupChannel` - Time-shifted replay of live channels
- **IPreRefreshProvider** (`XtreamVodProvider.cs`) - Enriches VOD metadata during refresh
- **IHasWebPages** (`Plugin.cs`) - Serves configuration UI pages

Service registration happens in `PluginServiceRegistrator.cs` using dependency injection.

### Directory Structure

- **Api/** - REST endpoints for configuration UI (`XtreamController.cs`)
- **Client/** - Xtream API abstraction layer
  - `IXtreamClient` interface and `XtreamClient` implementation
  - JSON models in `Client/Models/`
  - Custom JSON converters for API quirks
- **Configuration/** - Plugin settings and persistence
  - `PluginConfiguration.cs` - Main config (credentials, overrides, filtering)
  - `SerializableDictionary` - Custom XML-serializable dictionary
- **Providers/** - Metadata enhancement (`XtreamVodProvider`)
- **Service/** - Core business logic
  - `StreamService` - Stream metadata assembly and GUID-based hierarchy
  - `Restream` - Multi-client buffering for live streams
  - `TaskService` - Jellyfin task management integration

### Content Type Architecture

All content follows a **Category → Stream** hierarchy (Series adds Season/Episode levels):

1. **Live TV**: `LiveTvService` implements `ILiveTvService`
   - Fetches channels via `XtreamClient.GetLiveStreamsAsync()`
   - Applies category filtering and channel overrides
   - Parses stream names to extract tags like `[HD]`, `| FR |` using regex
   - Serves EPG data with 10-minute caching
   - Streams via `Restream` component (see below)

2. **VOD**: `VodChannel` implements `IChannel`
   - Categories as folders, streams as playable items
   - Metadata enrichment via `XtreamVodProvider` (plot, TMDB ID, genres)
   - Supports TMDB override for better metadata matching

3. **Series**: `SeriesChannel` implements `IChannel`
   - Hierarchy: Category → Series → Season → Episode
   - Data from `GetSeriesStreamsBySeriesAsync()` returns episodes grouped by season
   - Each level represented as folder in Jellyfin

4. **Catch-up**: `CatchupChannel` implements `IChannel`
   - Lists channels with `TvArchive` flag enabled
   - Creates day selector (based on `TvArchiveDuration`)
   - Uses timeshift API: `/streaming/timeshift.php?start=YYYY-MM-DD:HH:mm&duration=MINUTES`

### Streaming Components

**Restream** (`Service/Restream.cs`):
- Implements `ILiveStream` and `IDirectStreamProvider`
- Opens single HTTP connection to upstream Xtream URL
- Buffers into `WrappedBufferStream` (16MB ring buffer)
- Multiple clients read via `WrappedBufferReadStream` (independent read positions)
- Key benefit: Shares one upstream connection among multiple concurrent viewers

**StreamService** (`Service/StreamService.cs`):
- Builds `MediaSourceInfo` objects with stream URLs and codec metadata
- **GUID encoding**: Packs hierarchical IDs into GUIDs for stateless navigation
  - `ToGuid(prefix, int1, int2, int3)` creates GUID from 4 integers
  - `FromGuid(guid)` extracts the integers back
  - Prefix constants identify content type (Category, Stream, Series, Season, Episode, etc.)
- **Name parsing**: `ParseName()` extracts tags from stream names
  - Supports ASCII pipes `|TAG|`, brackets `[TAG]`, and Unicode variants (`│`, `┃`, `｜`)
  - Handles international provider naming conventions

### Xtream API Client

**XtreamClient** (`Client/XtreamClient.cs`):
- All requests via `QueryApi<T>()` helper
- Base URL pattern: `/player_api.php?username=X&password=Y&action=ACTION`
- Methods organized by content type:
  - Live: `GetLiveStreamsAsync()`, `GetLiveCategoryAsync()`
  - VOD: `GetVodCategoryAsync()`, `GetVodStreamsByCategoryAsync()`, `GetVodInfoAsync()`
  - Series: `GetSeriesCategoryAsync()`, `GetSeriesByCategoryAsync()`, `GetSeriesStreamsBySeriesAsync()`
  - EPG: `GetEpgInfoAsync()`

**JSON Converters** (in `Client/`):
- `StringBoolConverter` - Handles booleans returned as strings
- `SingularToListConverter` - Unwraps single objects into lists
- `Base64Converter` - Decodes base64 fields
- `OnlyObjectConverter` - Extracts object from wrapper array

### Configuration Filtering

User selections stored as:
```csharp
SerializableDictionary<int, HashSet<int>> LiveTv   // categoryId → streamIds
SerializableDictionary<int, HashSet<int>> Vod      // categoryId → streamIds
SerializableDictionary<int, HashSet<int>> Series   // categoryId → seriesIds
```

- Empty set = include all items in category
- Non-empty set = include only specified items
- Logic: `StreamService.IsConfigured()`

### Task Integration

`TaskService` uses reflection to invoke Jellyfin's internal scheduled tasks:
- `CancelIfRunningAndQueue()` triggers TV guide and channel refresh
- Called on configuration updates to refresh content immediately
- Avoids direct coupling to Jellyfin internals

## Important Patterns

### Tag Parsing Edge Cases

When working with stream name parsing in `StreamService.ParseName()`:
- The regex supports three pipe styles: ASCII `|`, Unicode `│` (U+2502), `┃` (U+2503), and `｜` (U+FF5C)
- Recent fix: Unicode pipe pattern changed from `[|│┃｜]` to `(?:|│┃｜)` to handle regex parsing correctly
- Tags can appear in brackets `[TAG]` or pipes `|TAG|`

### GUID Encoding Pattern

The hierarchy encoding in `StreamService.ToGuid()` packs 4 integers into a GUID:
```csharp
ToGuid(StreamType.VodCategory, categoryId, 0, 0)  // Category folder
ToGuid(StreamType.VodStream, categoryId, streamId, 0)  // VOD item
ToGuid(StreamType.SeriesSeason, seriesId, seasonId, 0)  // Season folder
ToGuid(StreamType.SeriesEpisode, seriesId, seasonId, episodeId)  // Episode item
```

When implementing new features that navigate hierarchies, use this pattern for stateless IDs.

### Configuration Update Flow

When `Plugin.UpdateConfiguration()` is called:
1. User agent is updated if changed
2. Two Jellyfin tasks are triggered via `TaskService`:
   - `Jellyfin.LiveTv.Guide.RefreshGuideScheduledTask` - Updates EPG and channel list
   - `Jellyfin.LiveTv.Channels.RefreshChannelsScheduledTask` - Refreshes channel entries
3. This ensures UI changes are immediately reflected without manual refresh

### Security Consideration

Per README: Jellyfin exposes stream URLs in API responses, and Xtream URLs include credentials in the path. This means anyone with library access can extract the Xtream username/password. This is a known limitation documented in "Known problems" section.

## UI Configuration

Web configuration pages located in `Configuration/Web/`:
- `XtreamCredentials.html/js` - API credentials entry
- `XtreamLive.html/js` - Live TV category/channel selection
- `XtreamLiveOverrides.html/js` - Channel number/name/logo overrides
- `XtreamVod.html/js` - VOD category/stream selection
- `XtreamSeries.html/js` - Series category/show selection
- `Xtream.css/js` - Shared styles and utilities

Backend API endpoints in `Api/XtreamController.cs` serve category/stream data to UI.

## Plugin Metadata

Defined in `build.yaml`:
- Plugin GUID: `5d774c35-8567-46d3-a950-9bb8227a0c5d`
- Target Jellyfin ABI: `10.11.0.0`
- Framework: `net9.0`
- Category: `LiveTV`

## License

GPLv3 - See LICENSE file. All source files include GPL header.
