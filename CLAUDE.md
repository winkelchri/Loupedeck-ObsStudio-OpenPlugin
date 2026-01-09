# CLAUDE.md - OBS Studio Loupedeck Plugin

This document provides comprehensive guidance for working with the OBS Studio Loupedeck Plugin codebase.

## Project Overview

This is a C# .NET 8.0 Loupedeck plugin that provides OBS Studio control through Loupedeck devices. It communicates with OBS via WebSocket (obs-websocket-dotnet library) and supports OBS Studio 28+ with Loupedeck 5.4+.

**Supported Devices:** LoupedeckCtFamily (CT, Live, Live S, Razer Stream Controller)

## Quick Start

### Prerequisites
- Visual Studio 2022
- .NET 8.0 SDK
- OBS Studio 28+ with WebSocket server enabled
- Loupedeck software installed (for testing)

### Build
```bash
# Restore packages and build
dotnet restore ObsStudioPlugin.sln
dotnet build ObsStudioPlugin.sln -c Release

# Or use Visual Studio
# Open ObsStudioPlugin.sln and build (Ctrl+Shift+B)
```

### Build Configurations
- `Debug` / `Release` - Windows builds (output: `bin/{config}/win/`)
- `Debug-Mac` / `Release-Mac` - macOS builds (output: `bin/{config}/mac/`)

### Testing
The plugin must be tested with actual Loupedeck hardware and OBS Studio running:
1. Build the plugin
2. Post-build scripts automatically create a plugin link file
3. Restart Loupedeck software
4. The plugin appears in Loupedeck's plugin list

## Project Structure

```
ObsStudioPlugin/
├── src/ObsPlugin/
│   ├── Actions/              # Loupedeck action implementations (19 commands)
│   ├── Proxy/                # OBS WebSocket communication layer (12 partial classes)
│   ├── Support/              # Helper classes and data structures
│   ├── Icons/                # SVG and PNG icon resources
│   ├── metadata/             # Plugin manifest (LoupedeckPackage.yaml)
│   ├── Sdk/net8/             # Loupedeck SDK DLLs
│   ├── ObsStudioPlugin.cs    # Main plugin entry point
│   ├── ObsStudioApplication.cs # OBS detection logic
│   └── ObsAppProxy.cs        # Proxy base class
├── ObsStudioPlugin.sln       # Solution file
├── azure-pipelines.yml       # CI/CD configuration
└── README.md
```

## Architecture

### Core Components

1. **ObsStudioPlugin.cs** - Main plugin class
   - Extends `Plugin` from Loupedeck SDK
   - Manages lifecycle: Load/Unload, application start/stop
   - Handles connection status reporting

2. **ObsStudioApplication.cs** - Application detection
   - Windows: Detects via registry `SOFTWARE\OBS Studio`
   - Mac: Detects via bundle `com.obsproject.obs-studio`
   - Supports both installed and portable OBS

3. **ObsAppProxy.cs** - WebSocket proxy (partial class)
   - Extends `OBSWebsocketDotNet.OBSWebsocket`
   - Split into domain-specific partial classes for maintainability

### Proxy Partial Classes

| File | Responsibility |
|------|----------------|
| `ObsAppProxy.cs` | Base class, lifecycle, event registration |
| `ObsAppProxy.Scenes.cs` | Scene selection and monitoring |
| `ObsAppProxy.Sources.cs` | Scene items and visibility |
| `ObsAppProxy.Audio.cs` | Audio sources, volume, mute |
| `ObsAppProxy.Filters.cs` | Source filter operations |
| `ObsAppProxy.Streaming.cs` | Stream control |
| `ObsAppProxy.Recording.cs` | Recording control |
| `ObsAppProxy.ReplayBuffer.cs` | Replay buffer |
| `ObsAppProxy.VirtualCam.cs` | Virtual camera |
| `ObsAppProxy.StudioMode.cs` | Studio mode |
| `ObsAppProxy.SceneCollections.cs` | Scene collections |
| `ObsAppProxy.Misc.cs` | Screenshots, utilities |

### Action Base Classes

```
PluginDynamicCommand             # Simple button actions
PluginTwoStateDynamicCommand     # Binary toggles
└── GenericOnOffSwitch           # Custom base for all toggles
PluginMultistateDynamicCommand   # Multistate selection (scenes, sources)
PluginDynamicAdjustment          # Dial/slider controls
ActionEditorCommand              # Actions with custom UI editors
```

### Key Patterns

**GenericOnOffSwitch** - Custom abstraction for toggle commands:
```csharp
internal class StreamingToggleCommand : GenericOnOffSwitch
{
    public StreamingToggleCommand()
        : base(
            name: "ToggleStreaming",
            displayName: "Streaming Toggle",
            offStateName: "Start streaming",
            onStateName: "Stop streaming",
            offStateImage: "StreamingToggleOn.svg",
            onStateImage: "StreamingToggleOff.svg")
    { }

    protected override void ConnectAppEvents(
        EventHandler<EventArgs> eventSwitchedOff,
        EventHandler<EventArgs> eventSwitchedOn)
    {
        ObsStudioPlugin.Proxy.AppEvtStreamingOff += eventSwitchedOn;
        ObsStudioPlugin.Proxy.AppEvtStreamingOn += eventSwitchedOff;
    }

    protected override void RunCommand(TwoStateCommand command)
    {
        switch (command)
        {
            case TwoStateCommand.TurnOff:
                ObsStudioPlugin.Proxy.AppStopStreaming();
                break;
            case TwoStateCommand.TurnOn:
                ObsStudioPlugin.Proxy.AppStartStreaming();
                break;
            case TwoStateCommand.Toggle:
                ObsStudioPlugin.Proxy.AppToggleStreaming();
                break;
        }
    }
}
```

**Parameter Encoding** - Hierarchical keys for dynamic parameters:
- `SceneKey`: `Collection||~~(%)~~||Scene`
- `SceneItemKey`: `Collection||~~(%)~~||Scene||~~(%)~~||SourceId||~~(%)~~||SourceName`

## Implemented Actions

### Toggle Commands
- `StreamingToggleCommand` - Start/stop streaming
- `RecordingToggleCommand` - Start/stop recording
- `RecordingPauseToggleCommand` - Pause/resume recording
- `ReplayBufferToggleCommand` - Enable/disable replay buffer
- `VirtualCameraToggleCommand` - Toggle virtual camera
- `StudioModeToggleCommand` - Toggle studio mode

### Selection Commands
- `SceneSelectCommand` - Switch scenes
- `SceneCollectionSelectCommand` - Switch scene collections
- `SourceVisibilityCommand` - Show/hide sources
- `SourceMuteCommand` - Mute/unmute audio
- `SourceFilterCommand` - Toggle source filters
- `GlobalAudioFilterCommand` - Toggle global audio filters

### Other Commands
- `SourceVolumeAdjustment` - Volume control (dial support)
- `TransitionCommand` - Execute transition
- `ReplayBufferSaveCommand` - Save replay buffer
- `ScreenshotCommand` - Take screenshot
- `UniversalStateSwitch` - Generic state setter
- `SourceVisibilitySwitch` - Set source visibility

## OBS Connection

### Configuration Location
- **Windows**: `%APPDATA%\obs-studio\plugin_config\obs-websocket\config.json`
- **Mac**: `~/Library/Application Support/obs-studio/plugin_config/obs-websocket/config.json`

### Connection Flow
1. Plugin detects OBS installation via registry/bundle
2. Reads WebSocket config from JSON
3. Waits for port availability (20 retries, 1s intervals)
4. Connects to `ws://127.0.0.1:{port}` with password
5. Subscribes to OBS events
6. Initializes state cache

### Error Handling Pattern
```csharp
// Safe execution wrapper
Helpers.TryExecuteSafe(() => this.SomeOperation());

// With return value
if (Helpers.TryExecuteFunc(() => this.GetValue(), out var result))
{
    // Use result
}

// Logging
this.Plugin.Log.Info("Operation succeeded");
this.Plugin.Log.Warning("Non-critical issue");
this.Plugin.Log.Error($"Critical failure: {ex.Message}");
```

## Dependencies

### NuGet Packages
- `obs-websocket-dotnet` (5.0.0.3) - OBS WebSocket client
- `Newtonsoft.Json` (13.0.3) - JSON serialization
- `System.Reactive` (4.3.2) - Reactive extensions
- `Websocket.Client` (4.4.43) - WebSocket support

### Loupedeck SDK (in `Sdk/net8/`)
- `PluginApi.dll` - Core plugin API
- `LoupedeckShared.dll` - Shared utilities

## Best Practices

### Adding a New Toggle Action
1. Create class extending `GenericOnOffSwitch`
2. Define constructor with name, display name, state names, icons
3. Implement `ConnectAppEvents()` to subscribe to proxy events
4. Implement `RunCommand()` to handle toggle/on/off
5. Add corresponding proxy methods if needed
6. Add icon SVGs to `Icons/` folder

### Adding a New Multistate Action
1. Create class extending `PluginMultistateDynamicCommand`
2. Override `OnLoad()` to subscribe to proxy events
3. Override `OnUnload()` to unsubscribe
4. Implement state management methods
5. Use `SceneKey` or similar for parameter encoding

### Adding Proxy Functionality
1. Add methods to appropriate partial class
2. Subscribe to OBS WebSocket events
3. Create custom events (`AppEvt...`) for actions to consume
4. Update state cache as needed
5. Use `Helpers.TryExecuteSafe()` for error handling

### Code Style
- Use `internal` access for plugin classes
- Follow existing naming: `App...` prefix for proxy methods/events
- Keep actions focused - one responsibility per class
- Use partial classes to organize large classes by domain
- Icons: SVG format preferred, 256x256 PNG for main icon

## Debugging

### Logs
- Plugin logs via `this.Plugin.Log.*`
- Check Loupedeck's log directory for plugin logs

### Common Issues
- **Plugin not loading**: Check SDK version compatibility
- **No connection**: Verify OBS WebSocket is enabled in OBS settings
- **Actions not appearing**: Ensure class is in `Loupedeck.ObsStudioPlugin.Actions` namespace

### Plugin Status States
- **Normal**: Connected to OBS
- **Warning**: OBS installed but not connected
- **Error**: OBS not installed or config invalid

## CI/CD

Azure Pipelines configuration (`azure-pipelines.yml`):
- Trigger: Push to main branch
- Build: Windows, Release configuration
- Tests: VSTest runner

## Version Info

- Plugin Version: 6.0.3 (metadata)
- Assembly Version: 5.11.0
- Target Framework: .NET 8.0
- Minimum Loupedeck Version: 6.0
- License: MIT
