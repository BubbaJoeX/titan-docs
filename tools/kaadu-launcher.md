# Kaadu Launcher

The **Kaadu** launcher (`Kaadu.exe`) is the official Windows client for **SWG: Titan**. It lives in the monorepo at `support/launcher_redux/`.

| Document | Audience |
|----------|----------|
| [Player guide](../../support/launcher_redux/README.md) | Install, play, troubleshooting |
| [Documentation index](../../support/launcher_redux/docs/README.md) | All developer/operator docs |
| [Architecture](../../support/launcher_redux/docs/architecture.md) | WPF app structure, services, update flow |
| [Build and deploy](../../support/launcher_redux/docs/build-and-deploy.md) | Compile, manifests, SFTP publish |
| [Administration](../../support/launcher_redux/docs/administration.md) | CDN layout, patch workflow |

## Summary

- **Stack:** .NET 9 WPF, x64 single-file publish
- **Game binary:** `SwgTitan_r.exe` with manifest-based install from SFTP
- **Tools:** `kaadu-manifest` (manifest generator), `Kaadu.Deploy` (launcher upload)

See the [documentation index](../../support/launcher_redux/docs/README.md) for paths, config constants, and related Titan directories.
