# FileGrid Plugin Development & SDK Standard Manual (v2.0)

Welcome to the **FileGrid Plugin Development Guide**. FileGrid features a high-performance, modular plugin architecture designed for speed, safety, and seamless UX.

---

## 1. Architecture Overview

FileGrid supports two distinct plugin execution models:

| Execution Model | Description | Best Suited For | Isolation Level |
| :--- | :--- | :--- | :--- |
| **`InProcessUI`** | WinUI 3 XAML Control loaded dynamically. Runs inside the FileGrid host process for instant rendering, smooth animations, and Pearl Theme synchronization. | Batch renaming, file taggers, quick inspectors, side panels | Host-managed (Circuit Breaker protected) |
| **`StandaloneApp`** | Independent executable process (`.exe`) launched with target file paths. Bound to a Windows Job Object (`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`) to prevent zombie processes. | Media players, 3D model viewers, heavy converters | Full Process Sandbox |

---

## 2. Plugin Structure & Package Format (`.fgpkg`)

An FileGrid plugin package (`.fgpkg`) is a standard ZIP archive containing:

```text
MyPlugin.fgpkg
 ├── plugin.json              # Mandatory: Plugin Manifest
 ├── icon.png                 # Optional: 32x32 or 64x64 icon
 ├── MyPlugin.dll             # Primary binary (or .exe for Standalone)
 └── assets/                  # Additional assets, localization, or native DLLs
```

---

## 3. Manifest Specification (`plugin.json`)

Every plugin **MUST** contain a `plugin.json` file in its root directory:

```json
{
  "id": "com.yourorg.hashchecker",
  "name": "Checksum & Hash Calculator",
  "version": "1.0.0",
  "minAppVersion": "1.0.0",
  "description": "Calculates MD5, SHA-1, SHA-256, and CRC32 hashes with copy-to-clipboard.",
  "author": "Your Name / Organization",
  "enabled": true,
  "type": "InProcessUI",
  "entryPoint": "HashChecker.dll",
  "entryClass": "YourNamespace.HashCheckerControl",
  "uiTargets": ["ContextMenu", "Toolbar"],
  "handleExtensions": {
    "AllFiles": [".*"]
  },
  "permissions": ["FileSystem.Read"]
}
```

### Manifest Fields Reference

| Field | Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `id` | `string` | **Yes** | Unique identifier (e.g., reverse domain: `com.developer.toolname`). |
| `name` | `string` | **Yes** | Display name shown in UI and Plugins Hub. |
| `version` | `string` | **Yes** | Semantic version string (e.g. `1.0.0`). |
| `minAppVersion` | `string` | No | Minimum FileGrid version required. |
| `description` | `string` | No | Short description of plugin capabilities. |
| `author` | `string` | No | Author or organization name. |
| `type` | `string` | **Yes** | `"InProcessUI"` or `"StandaloneApp"`. |
| `entryPoint` | `string` | **Yes** | Relative path to main `.dll` or `.exe`. |
| `entryClass` | `string` | For UI | Full C# class name implementing the UI Control. |
| `uiTargets` | `array` | No | UI entry points: `["ContextMenu", "Toolbar", "SidePane"]`. |
| `handleExtensions`| `object` | No | Mapping of file categories to extensions (e.g. `{ "Images": [".png", ".jpg"] }` or `{ "All": [".*"] }`). |
| `permissions` | `array` | No | Requested capabilities shown during install (e.g. `["FileSystem.Read", "Network.Access"]`). |

---

## 4. UI Extension Points (`uiTargets`)

Plugins can declare where they appear in FileGrid:

1. **`"ContextMenu"`**:
   - Appears in the file right-click menu when compatible files are selected.
   - If more than 3 plugins target `ContextMenu`, FileGrid automatically groups them into a tidy `Plugins ▶` submenu to keep the context menu uncluttered.
2. **`"Toolbar"`**:
   - Surfaces as a quick-action icon button on the active window toolbar.
3. **`"SidePane"`**:
   - Embeds as a dockable side pane panel.

---

## 5. Storage & Lifecycle Guidelines

### 5.1 Code vs User Data Separation
* **Plugin Binaries (Read-Only):**
  - System plugins: `<AppDirectory>\Plugins\<PluginId>\`
  - User plugins: `%LOCALAPPDATA%\FileGrid\Plugins\<PluginId>\`
  *Never write persistent user data into the plugin binary directory!*
* **Plugin Data & Settings (Read-Write):**
  - Always write user presets, databases, and cached settings to:
    `%LOCALAPPDATA%\FileGrid\PluginData\<PluginId>\`
  - This ensures user settings are preserved across plugin updates!

### 5.2 Non-Blocking File Sharing
When opening files selected by the user, **ALWAYS** allow shared reading and deletion:
```csharp
using var stream = new FileStream(
    filePath, 
    FileMode.Open, 
    FileAccess.Read, 
    FileShare.ReadWrite | FileShare.Delete
);
```

### 5.3 Cancellation & Streaming for Large Selections
If the user selects 50,000 files, your plugin should process items asynchronously with a `CancellationToken` rather than freezing the UI thread:
```csharp
public async Task ProcessFilesAsync(IReadOnlyList<string> files, CancellationToken cancellationToken)
{
    foreach (var file in files)
    {
        cancellationToken.ThrowIfCancellationRequested();
        await ProcessSingleFileAsync(file);
    }
}
```

---

## 6. Packaging Your Plugin (`.fgpkg`)

FileGrid provides an automated packaging tool:

```powershell
.\tools\pack-plugin.ps1 -SourceDir ".\Plugins\MyPlugin" -OutputDir ".\Dist"
```

The packager validates:
- Manifest JSON schema and required fields.
- Binary existence and generates a SHA-256 hash.
- Compresses files with optimal Deflate compression into `.fgpkg`.

---

## 7. Installing and Testing

1. **In-App Installation:**
   - Open **Settings** (gear icon) -> **Plugins**.
   - Click **"Install Plugin (.fgpkg)..."** or simply drag & drop your `.fgpkg` into the drop zone.
   - Review the permission confirmation dialog and click **Install**.
2. **Testing in Developer Mode (Unpacked):**
   - Place your plugin folder directly in `%LOCALAPPDATA%\FileGrid\Plugins\<PluginId>\` or `<AppDirectory>\Plugins\<PluginId>\`.
   - FileGrid detects it immediately without packaging.
3. **Uninstallation:**
   - In Settings -> Plugins, user-installed plugins display an **Uninstall** button.
   - Choose whether to retain or purge plugin user data during uninstallation.
