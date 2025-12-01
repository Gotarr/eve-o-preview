# EVE-O Preview AI Coding Agent Instructions

## Project Overview
EVE-O Preview is a **Windows Forms/.NET 8.0 desktop application** that provides live thumbnail previews of multiple EVE Online client windows. It uses Windows DWM (Desktop Window Manager) APIs to create live preview thumbnails for fast task switching between game clients. The codebase supports **both Windows and Linux (Wine)** through conditional compilation using the `EVEOTARGET` build property.

## Architecture Patterns

### Dependency Injection & IoC
- **LightInject** container wraps DI, accessed through custom `IIocContainer` interface
- All services registered as **singletons** in `Program.InitializeApplicationController()`
- Registration pattern: `container.Register<IService>()` for auto-discovery, `container.Register<IService, Implementation>()` for explicit mapping
- Views registered via `controller.RegisterView<IMainFormView, MainForm>()`

### MVP (Model-View-Presenter) Pattern
- Presenters inherit from `Presenter<TView>` or `PresenterGeneric<TView, TArgument>`
- Views implement interfaces (e.g., `IMainFormView`, `IThumbnailView`) and use **action delegates** for event callbacks
- Example: `this.View.FormActivated = this.Activate;` in `MainFormPresenter.cs`
- Presenters are created via `controller.Run<MainFormPresenter>()` or `controller.Create<TService>()`

### MediatR Event-Driven Communication
- **MediatR** (v9.0.0) handles cross-component messaging
- Handlers in `Mediator/Handlers/{Configuration|Services|Thumbnails}/`
- Notifications: `INotificationHandler<ThumbnailListUpdated>` for broadcast events
- Requests: `IRequestHandler<StartService>` for command-style operations
- Messages defined in `Mediator/Messages/` hierarchy

### Dual Thumbnail Rendering Strategies
- **LiveThumbnailView**: Uses DWM API (`IDwmThumbnail`) for real-time preview (Windows-only, low memory)
- **StaticThumbnailView**: Screenshot-based fallback for compatibility mode (works in RDP, higher memory ~180MB vs ~50MB)
- Selection logic in `ThumbnailManager.cs`, controlled by `CompatibilityMode` config option

## Critical Cross-Platform Considerations
- Conditional compilation: `#if LINUX` / `#if WINDOWS` extensively used in:
  - `Services/Implementation/WindowManager.cs` (DWM availability checks)
  - `Services/Implementation/ThumbnailManager.cs` (refresh rates, client handling)
  - `Configuration/Implementation/ThumbnailConfiguration.cs` (default settings)
- Build command examples in README.md:
  ```
  dotnet build src\Eve-O-Preview\Eve-O-Preview.csproj --configuration Release -p:EVEOTARGET="Linux" -p:AssemblyVersion="8.0.2.0"
  dotnet build src\Eve-O-Preview\Eve-O-Preview.csproj --configuration Release -p:EVEOTARGET="Windows" -p:AssemblyVersion="8.0.2.0"
  ```

## Configuration System
- **JSON-based** storage in `EVE-O-Preview.json` (next to executable)
- Loaded/saved via `ConfigurationStorage.cs` using **Newtonsoft.Json**
- Two config objects: `IAppConfig` (app-level) and `IThumbnailConfiguration` (feature-rich)
- Per-client overrides: `PerClientActiveClientHighlightColor`, `PerClientThumbnailSize`, `PerClientZoomAnchor`, etc.
- Hotkeys defined in config: `ClientHotkey` (per-client), `CycleGroup1-5ForwardHotkeys`, `MinimizeAllClientsHotkeys`
- Validation via `ApplyRestrictions()` method after loading

## Key Services & Responsibilities
- **IThumbnailManager**: Core orchestrator managing thumbnail lifecycle, hotkey handling, client activation
- **IWindowManager**: Windows API wrapper for DWM composition, window enumeration, minimize/restore
- **IProcessMonitor**: Tracks running EVE client processes (filtered by `ExecutablesToPreview` config list)
- **IThumbnailViewFactory**: Factory creating `LiveThumbnailView` or `StaticThumbnailView` based on DWM availability

## Interop & Native Code
- P/Invoke wrappers in `Services/Interop/`:
  - `DwmNativeMethods.cs` - DWM thumbnail APIs
  - `User32NativeMethods.cs` - Window management
- Handle COM exceptions gracefully (DWM can become unavailable during user account switches)
- Exception types to catch: `ArgumentException`, `COMException` in thumbnail operations

## Development Workflows

### Building Locally
- Use Visual Studio 2022 or `dotnet build` CLI
- Solution file: `src/EVE-O-Preview.sln` (contains Eve-O-Preview + Eve-O-Mock projects)
- Output paths differ by configuration:
  - Debug: `src\bin`
  - Release: `bin\` (repository root)
- Eve-O-Mock: Simple WPF testing harness for simulating EVE windows

### Release Process
- Automated via GitHub Actions (`.github/workflows/release.yml`)
- Builds triggered on **release tag creation** (e.g., `v8.0.2.0`)
- Matrix strategy builds both Windows and Linux variants
- Linux build uses `--self-contained true`, Windows `--self-contained false`
- Outputs: `Release-{version}-{platform}.zip` uploaded as release assets

### Debugging Tips
- Single instance enforcement via named Mutex (`EVE-O-Preview Single Instance Mutex`)
- Exception handling setup in `ExceptionHandler.SetupExceptionHandlers()`
- Config file must be writable (don't install to Program Files)
- DWM issues: Check `IWindowManager.IsCompositionEnabled` property

## Common Coding Patterns

### Adding New Configuration Options
1. Add property to `IThumbnailConfiguration` interface
2. Implement with `[JsonProperty]` in `ThumbnailConfiguration.cs`
3. Add validation in `ApplyRestrictions()` if needed
4. Reference in UI via `MainFormPresenter._configuration.YourOption`
5. Save via `_configurationStorage.Save()` after changes

### Adding New Hotkeys
1. Define hotkey list property in `IThumbnailConfiguration` (e.g., `List<string> NewFeatureHotkeys`)
2. Register handler in `ThumbnailManager` constructor: `RegisterYourFeatureHotkey(config.NewFeatureHotkeys?.Select(x => config.StringToKey(x)))`
3. Use `HotkeyHandler` class from `Hotkeys/HotkeyHandler.cs`
4. Implement callback logic for hotkey activation

### Adding MediatR Messages
1. Create message class in `Mediator/Messages/{Category}/YourMessage.cs` (implement `INotification` or `IRequest`)
2. Create handler in `Mediator/Handlers/{Category}/YourMessageHandler.cs`
3. Publish: `await _mediator.Publish(new YourMessage())`
4. Send: `await _mediator.Send(new YourRequest())`

## Important Constraints
- **No testing framework** present - validate manually or add tests if needed
- **No EVE Online interaction** beyond window management (strictly enforced per EULA/ToS)
- Must support **fixed window and window mode** only (not fullscreen)
- Configuration changes while app is closed to prevent overwrites
- Avoid `Program Files` installation due to config write requirements

## File Naming & Namespace Conventions
- Namespace: `EveOPreview` (main), `EveOPreview.{Services|Configuration|View|Presenters|Mediator}`
- Interfaces prefixed with `I` (e.g., `IWindowManager`)
- Implementations in `Implementation/` subfolders, interfaces in `Interface/` subfolders
- Partial classes for WinForms: `{ClassName}.cs` (logic) + `{ClassName}.Designer.cs` (generated) + `{ClassName}.resx` (resources)

## References to Study
- Core entry point: `src/Eve-O-Preview/Program.cs`
- Main orchestrator: `src/Eve-O-Preview/Services/Implementation/ThumbnailManager.cs` (~1057 lines)
- Configuration loading: `src/Eve-O-Preview/Configuration/Implementation/ConfigurationStorage.cs`
- DWM interaction: `src/Eve-O-Preview/Services/Implementation/DwmThumbnail.cs`
- README.md contains comprehensive user documentation and config examples
