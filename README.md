# MOS GoSHS Plugin

MOS plugin for [GoSHS](https://github.com/goshs-labs/goshs), a feature-rich
single-binary file server.

The plugin is intentionally focused on the common MOS use case: expose a
directory such as `/mnt/user/binaries` over HTTP while keeping the full GoSHS
CLI available for advanced or one-off usage.

## Dashboard

The plugin page provides:

- GoSHS install/update status
- start, stop, and restart controls
- webroot path
- listen IP and HTTP port
- auto-start
- read-only mode
- chat disable toggle
- optional WebDAV and WebDAV port
- a shortcut to open the GoSHS web UI

## CLI

After the runtime is installed, the complete upstream binary is available as:

```bash
goshs --help
```

For example, a one-off server can still be launched directly:

```bash
goshs -d /mnt/user/something-else -p 9000 --webdav
```

The dashboard-managed instance uses `/usr/bin/goshs`; its PID is stored at
`/var/run/mos-goshs.pid` and its log is written to `/var/log/goshs.log`.

## Default configuration

```json
{
  "auto_start": true,
  "webroot": "/mnt/user/binaries",
  "listen_ip": "0.0.0.0",
  "port": 8000,
  "read_only": true,
  "no_chat": true,
  "webdav": false,
  "webdav_port": 8001
}
```

## Releases

Pushing a numeric tag such as `0.1.0` runs the GitHub Actions release workflow
and publishes the MOS plugin `.deb` plus its MD5 file.

The upstream GoSHS binary is not vendored into this repository. The plugin
downloads the latest release for the current MOS architecture when you click
Install/Update.