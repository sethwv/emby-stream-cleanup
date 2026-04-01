# Emby Stream Cleanup

A [Dispatcharr](https://github.com/Dispatcharr/Dispatcharr) plugin that monitors client activity and automatically terminates idle Emby connections in Dispatcharr.

## The Problem

When Emby connects to Dispatcharr for live TV, it opens client connections that persist even after users stop watching. Emby does not close these connections on its own, which wastes provider stream slots and can hit connection limits.

## How It Works

1. You tell the plugin how to identify Emby's connections (IP address, hostname, or XC username).
2. A background monitor polls all active Dispatcharr channels on a configurable interval (default: 10s).
3. For each channel, it checks whether clients matching the identifier are still receiving data.
4. If a matching client's `last_active` timestamp goes stale for longer than the idle timeout (default: 30s), the plugin terminates that specific connection via `ChannelService.stop_client()`.
5. When an Emby server URL is configured, the plugin also polls Emby's Sessions API to detect **orphaned** connections — streams that Emby considers stopped but Dispatcharr is still serving. These are terminated after confirmation over two consecutive poll cycles.

No webhooks, no external configuration in Emby. The plugin watches Dispatcharr's own activity data directly.

## Safety

- Only connections matching the configured Emby identifier are ever affected.
- Non-matching clients on the same channel are never touched.
- Active Emby connections (still receiving data) are never terminated.
- The plugin only reads Dispatcharr's existing client metadata from Redis; it does not modify any upstream state.

## Installation

1. Download the latest release zip from the [releases page](https://github.com/sethwv/emby-stream-cleanup/releases).
2. In Dispatcharr, go to **Plugins** and upload the zip.
3. Restart Dispatcharr.
4. Enable the plugin and configure settings.

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| Client Identifier | _(empty)_ | IP, hostname, or XC username Emby uses to connect. Comma-separated for multiple values. Use `ALL` to match every client. |
| Idle Timeout | `30` | Seconds a matching client must be idle before termination |
| Poll Interval | `10` | How often (seconds) to scan for idle clients |
| Emby Server URL | _(empty)_ | Base URL of the Emby server (e.g. `http://192.168.1.100:8096`). Enables orphan detection via Sessions API. |
| Emby API Key | _(empty)_ | API key for the Emby server (Settings > API Keys) |
| Enable Debug Server | `false` | Start an HTTP debug dashboard (optional) |
| Debug Server Port | `9193` | Port for the debug server |

### Finding Your Emby Identifier

Check Dispatcharr's active connections while Emby is streaming. The IP address or XC username shown for Emby's connection is what you enter here. Multiple values can be comma-separated (e.g. `192.168.1.100, emby-server`).

Hostnames are automatically resolved to IP addresses.

## Docker

If running in Docker, expose the debug server port if you want to use the dashboard:

```yaml
ports:
  - "9193:9193"
```

The default host binding is `0.0.0.0`, which works with Docker port mapping.

## Debug Dashboard

When the debug server is enabled, visit `http://<host>:9193/debug` to see:

- All active channels with connected clients
- Which clients match the Emby identifier (highlighted)
- Current idle duration per client
- Recent termination history

The page auto-refreshes at the poll interval rate.

## Plugin Actions

In the Dispatcharr plugin settings page:

- **Start Monitor** - Start the background activity monitor (and debug server if enabled)
- **Stop Monitor** - Stop the monitor and debug server
- **Check Status** - Show whether the monitor and debug server are running

## Requirements

- Dispatcharr v0.22.0 or later
- Plugin must be enabled in Dispatcharr

## Building from Source

```bash
./package.sh
```

This creates a versioned zip file ready to upload to Dispatcharr.
