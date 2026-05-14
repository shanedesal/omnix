# omnix

> Manage files across all your devices from one place — phone, PC, tablet. Works over local WiFi with automatic internet fallback.

---

## What is omnix?

omnix is a cross-platform Flutter app that turns any of your devices into a connected file network. One device acts as the **hub** — the control center where you can browse, transfer, rename, copy, and delete files on any other connected device. No cloud subscription required. No cables. No third-party storage.

When your devices are on the same WiFi, transfers are fast and fully local. When they're not (different networks, mobile data), omnix falls back to a relay through Supabase so you stay connected.

---

## Supported Platforms

| Platform | Role | File Access |
|----------|------|-------------|
| Android | Hub or node | Full storage via Storage Access Framework |
| Windows | Hub or node | Full file system via `dart:io` |
| iOS | Node | App sandbox + user-shared files |
| macOS | Hub or node | Full file system via `dart:io` |

---

## Core Features

- **Device management** — add devices by scanning a QR code, see all connected nodes in one screen
- **Unified file browser** — browse any connected device's files as if they were local
- **File operations** — copy, move, rename, delete across devices
- **Transfer queue** — background transfers with progress, pause, and resume
- **Auto network switching** — local WiFi first, internet relay fallback when needed
- **PIN protection** — lock the hub behind a PIN so only you can connect
- **Duplicate detection** — find and remove duplicate files across devices
- **Bulk rename** — rename files using pattern templates (e.g. `{name}_{date}_{index}`)

---

## Tech Stack

This section explains every package and tool used, why it was chosen, and exactly what it does in the app. Read this carefully before diving into the code.

---

### Flutter (framework)

**What it is:** Google's UI toolkit for building natively compiled apps from a single Dart codebase.

**Why we use it:** One codebase runs on Android, iOS, Windows, and macOS. The file management logic, UI, networking, and state management are all written once in Dart and compiled natively per platform. No React Native bridge, no web wrapper — real native performance.

**What it does here:** Everything. The entire app — UI screens, business logic, networking — is Flutter/Dart.

---

### `shelf` + `shelf_router` + `shelf_web_socket`

**What it is:** A composable Dart HTTP server library, similar to Express.js in Node.

**Why we use it:** omnix needs to run an HTTP server *inside* the app on each device so that other devices can connect to it and request file listings, downloads, and uploads. `shelf` is the only production-ready HTTP server library in Dart that works on both mobile and desktop.

**What it does here:**
- Each device runs a `shelf` server on port `8080` (configurable)
- `shelf_router` maps routes: `GET /files` returns a directory listing, `GET /download?path=...` streams a file, `POST /upload` receives a file
- `shelf_web_socket` upgrades HTTP connections to WebSocket for real-time events like transfer progress and remote file change notifications

```
Device A (hub)  ──HTTP/WebSocket──►  Device B (node, shelf server on :8080)
                ◄── file listing ──
                ──── download ────►
```

---

### `multicast_dns`

**What it is:** A Dart package for mDNS (multicast DNS), the same protocol used by AirDrop and Bonjour to find nearby devices on a local network.

**Why we use it:** Instead of making users manually type in an IP address to connect devices, omnix broadcasts each device's presence on the local network. The hub listens for these broadcasts and automatically discovers nearby nodes — no manual setup needed.

**What it does here:**
- When the app starts, it registers the device on the local network as `omnix-{deviceName}._tcp`
- The hub continuously scans for other `omnix-*` services on the network
- When a new device is found, it shows up in the hub's device list automatically
- When a device leaves the network, it's marked as offline

```
Phone starts app → broadcasts "omnix-myphone._tcp" on LAN
PC hub listens  → discovers "omnix-myphone" → shows it in device list
```

---

### Supabase Realtime

**What it is:** Supabase's managed WebSocket channel service, built on top of Phoenix Channels.

**Why we use it:** When two devices are on different networks (one on home WiFi, one on mobile data), they can't reach each other directly. Supabase Realtime acts as a relay — both devices connect to the same Supabase channel and tunnel data through it. We use the free tier which gives us 200 concurrent connections and 2 million messages/month (each message up to 256 KB).

**What it does here:**
- The hub creates a session channel with a unique ID (e.g. `omnix-session-abc123`)
- The remote node joins the same channel
- File chunks are sent as broadcast messages through the channel, base64-encoded in 256 KB pieces
- The hub reconstructs the chunks into the complete file on the receiving end
- This path is **only used when mDNS local discovery fails** — local WiFi transfers never touch Supabase

```
Hub ──► Supabase channel "omnix-session-abc123" ◄── Node
        (relay, chunks pass through, not stored)
```

**Free tier limits relevant to us:**
- 200 concurrent connections — fine for personal use
- 2 million messages/month @ 256 KB each — ~500 GB of relay transfer, more than enough
- Projects pause after 7 days of inactivity — not an issue for active use

---

### `supabase_flutter`

**What it is:** The official Supabase Flutter SDK.

**Why we use it:** Handles Supabase authentication, Realtime channel subscriptions, and the connection lifecycle. Saves us from writing raw WebSocket management code.

**What it does here:** Initializes the Supabase client with your project URL and anon key, manages the Realtime connection, and exposes the channel API we use for the relay tunnel.

---

### Riverpod

**What it is:** A compile-safe, reactive state management library for Flutter. The modern successor to Provider.

**Why we use it:** omnix has a lot of async, reactive state — device connection status, ongoing transfers, file listings that update when remote files change, network mode switching. Riverpod handles all of this cleanly without boilerplate, and its `AsyncNotifier` pattern maps perfectly to our use cases.

**What it does here:**
- `DeviceRegistryNotifier` — holds the list of connected devices and their status (online/offline, LAN/relay)
- `TransferQueueNotifier` — manages the queue of active and pending file transfers
- `FileListingNotifier` — fetches and caches the file tree for each connected device
- `NetworkModeNotifier` — tracks whether we're in LAN mode or relay mode per device

---

### `permission_handler`

**What it is:** A Flutter plugin that provides a unified API for requesting runtime permissions across Android and iOS.

**Why we use it:** Android requires explicit user permission to read and write external storage. On Android 11+, full file access requires the special `MANAGE_EXTERNAL_STORAGE` permission which opens a system settings page (not a normal dialog). `permission_handler` abstracts all of this.

**What it does here:**
- Requests `READ_EXTERNAL_STORAGE` / `WRITE_EXTERNAL_STORAGE` on Android < 11
- Requests `MANAGE_EXTERNAL_STORAGE` and opens the system settings page on Android 11+
- Checks permission status before any file operation and shows a rationale dialog if denied
- On iOS, handles photo library and file access permissions for the sandboxed environment

---

### `path_provider`

**What it is:** A Flutter plugin to find commonly used locations on the file system in a cross-platform way.

**Why we use it:** Paths like "downloads folder" or "documents folder" are different on every platform. `path_provider` abstracts this so the same Dart code works everywhere.

**What it does here:**
- Provides the app's local storage directory for saving transfer metadata and temporary file chunks
- Provides the downloads directory path where received files are saved by default
- Used as the default root when the user hasn't selected a specific folder to share

---

### `dart:io`

**What it is:** Dart's built-in library for file system access, sockets, HTTP, and processes.

**Why we use it:** On Windows and macOS, `dart:io` gives full, unrestricted file system access — no special packages needed. `File`, `Directory`, and `FileSystemEntity` are all we need to list, read, write, and delete files on desktop platforms.

**What it does here:**
- Powers the platform file adapter on Windows and macOS
- `Directory.list(recursive: true)` for building the file tree
- `File.openRead()` for streaming file bytes over the shelf server
- `File.writeAsBytes()` / `IOSink` for receiving uploaded files
- `FileStat` for file metadata (size, last modified, type)

---

### `flutter_foreground_task`

**What it is:** A Flutter plugin that runs a Dart callback in a foreground service on Android, keeping it alive even when the app is backgrounded.

**Why we use it:** Android aggressively kills background processes to save battery. If the hub or node app is backgrounded, Android would kill the `shelf` server and cut off all connections. A foreground service (shown as a persistent notification) prevents this.

**What it does here:**
- Runs the `shelf` HTTP server inside a foreground service on Android
- Shows a persistent "omnix is running" notification while the server is active
- Keeps mDNS broadcasting alive in the background
- Not needed on Windows/macOS where apps can run freely in the background

---

### `qr_flutter` + `mobile_scanner`

**What it is:** `qr_flutter` generates QR codes as Flutter widgets. `mobile_scanner` reads QR codes using the device camera.

**Why we use it:** Pairing two devices manually (typing IPs or session codes) is annoying. Scanning a QR code takes 2 seconds. The QR code encodes everything needed to connect: the Supabase session ID, the local IP address, and a one-time pairing token.

**What it does here:**
- Hub displays a QR code containing `{ ip: "192.168.1.5", port: 8080, sessionId: "abc123", token: "xyz" }`
- New device scans it with `mobile_scanner`, extracts the connection info, and initiates pairing
- The pairing token is validated and a trusted device entry is created on both ends
- After first pairing, devices remember each other and reconnect automatically

---

### `flutter_secure_storage`

**What it is:** A Flutter plugin that stores key-value data in the platform's secure enclave (Android Keystore / iOS Keychain / Windows DPAPI).

**Why we use it:** The hub PIN, pairing tokens, and Supabase session keys should not be stored in plain SharedPreferences. `flutter_secure_storage` encrypts them at rest using the platform's hardware security module.

**What it does here:**
- Stores the hub PIN hash (bcrypt)
- Stores trusted device tokens so paired devices reconnect without re-scanning
- Stores the Supabase anon key (though this is technically public, it's good practice)

---

### `crypto`

**What it is:** Dart's cryptography library providing MD5, SHA-1, SHA-256, and HMAC implementations.

**Why we use it:** Duplicate file detection works by comparing file hashes — two files with the same MD5 hash are (almost certainly) identical regardless of filename.

**What it does here:**
- Computes MD5 hashes of files for duplicate detection
- Hashes are computed in chunks so large files don't load fully into memory
- SHA-256 used for pairing token generation and validation

---

### `dio`

**What it is:** A powerful HTTP client for Dart with interceptors, progress tracking, and cancel tokens.

**Why we use it:** Flutter's built-in `http` package is fine for simple requests but lacks built-in transfer progress callbacks and cancellation. `dio` makes it easy to stream large file downloads with a progress indicator and cancel mid-transfer.

**What it does here:**
- Downloads files from remote devices' `shelf` servers with byte-level progress reporting
- Feeds progress into the `TransferQueueNotifier` so the UI shows a live progress bar
- Handles cancel tokens so the user can abort a transfer mid-way
- Used for the LAN path only — relay transfers go through Supabase Realtime

---

## Project Structure

```
lib/
├── core/
│   ├── network/
│   │   ├── lan_transport.dart        # shelf server + mDNS
│   │   └── relay_transport.dart      # Supabase Realtime tunnel
│   ├── file/
│   │   ├── file_adapter.dart         # abstract interface
│   │   ├── android_adapter.dart      # SAF implementation
│   │   ├── desktop_adapter.dart      # dart:io implementation
│   │   └── ios_adapter.dart          # sandboxed implementation
│   └── crypto/
│       └── hash_service.dart         # MD5 for duplicates, SHA-256 for tokens
├── data/
│   ├── models/
│   │   ├── device.dart
│   │   ├── file_item.dart
│   │   └── transfer.dart
│   └── repositories/
│       ├── device_repository.dart
│       └── file_repository.dart
├── providers/                        # all Riverpod providers
│   ├── device_registry.dart
│   ├── transfer_queue.dart
│   ├── file_listing.dart
│   └── network_mode.dart
└── ui/
    ├── devices/                      # device list + pairing screens
    ├── browser/                      # file browser screen
    ├── transfers/                    # transfer queue screen
    └── settings/                    # PIN, preferences
```

---

## Connection Flow

```
1. App starts
   └── Register on LAN via mDNS ("omnix-mydevice._tcp")
   └── Start shelf HTTP server on :8080
   └── Start foreground service (Android only)

2. Pair a new device
   └── Hub shows QR code with { ip, port, sessionId, token }
   └── New device scans QR → validates token → saved as trusted device

3. File operation (e.g. download)
   └── Check: is device reachable on LAN? (mDNS ping)
       ├── YES → dio HTTP GET to device's shelf server (fast, local)
       └── NO  → chunk file via Supabase Realtime broadcast (relay fallback)

4. Transfer completes
   └── TransferQueueNotifier updates UI
   └── File saved to destination path via platform adapter
```

---

## Environment Setup

Create a `.env` file (or configure via `--dart-define`):

```
SUPABASE_URL=https://yourproject.supabase.co
SUPABASE_ANON_KEY=your-anon-key
HUB_PORT=8080
```

---

## Build & Run

```bash
# Android
flutter run -d android

# Windows
flutter run -d windows

# macOS
flutter run -d macos

# iOS
flutter run -d ios
```

---

## Roadmap

- [x] Architecture design
- [ ] Phase 1 — LAN discovery + shelf server + basic file listing
- [ ] Phase 2 — hub UI + unified file browser across devices
- [ ] Phase 3 — file operations (copy, move, delete, rename)
- [ ] Phase 4 — Supabase relay fallback + QR pairing
- [ ] Phase 5 — Windows desktop node + background service
- [ ] Phase 6 — duplicate detection + bulk rename

---

## License

MIT