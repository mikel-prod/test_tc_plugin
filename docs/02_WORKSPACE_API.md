# Workspace API Reference

**AI Agent Instructions:** This document details the Workspace API capabilities, methods, events, and limitations.

---

## Loading the Workspace API

### CDN Import (Recommended)
```html
<script src="https://components.connect.trimble.com/trimble-connect-workspace-api/index.js"></script>
```

**Critical:** Must be loaded BEFORE your application code attempts to use it.

### Global Variable
**Name:** `TrimbleConnectWorkspace` (NOT `WorkspaceAPI`)

**Common Error:**
```javascript
// ❌ WRONG
if (typeof WorkspaceAPI !== 'undefined') { ... }

// ✅ CORRECT
if (typeof TrimbleConnectWorkspace !== 'undefined') { ... }
```

---

## Connection Establishment

### Method: `TrimbleConnectWorkspace.connect()`

**Signature:**
```typescript
connect(
  targetWindow: Window,
  eventHandler: (event: string, args: any) => void,
  timeout?: number
): Promise<API>
```

**Parameters:**
- `targetWindow`: `window.parent` (plugin runs in iframe)
- `eventHandler`: Callback function for all events
- `timeout`: Connection timeout in milliseconds (default: 30000)

**Returns:** Promise resolving to API object

**Example:**
```javascript
const API = await TrimbleConnectWorkspace.connect(
    window.parent,
    (event, args) => {
        console.log('Event received:', event, args);

        switch (event) {
            case "extension.command":
                handleCommand(args.data);
                break;
            case "extension.accessToken":
                handleTokenUpdate(args.data);
                break;
            case "extension.userSettingsChanged":
                handleSettingsChange();
                break;
            default:
                console.log('Unknown event:', event);
        }
    },
    30000 // 30 second timeout
);
```

**Error Handling:**
```javascript
try {
    const API = await TrimbleConnectWorkspace.connect(window.parent, eventHandler, 30000);
    console.log('Connected successfully');
} catch (error) {
    console.error('Connection failed:', error);
    // Common causes:
    // 1. Not running inside Trimble Connect iframe
    // 2. Network timeout
    // 3. Workspace API script failed to load
}
```

---

## API Namespaces

The API object contains these namespaces:

```javascript
API = {
    project: ProjectAPI,
    user: UserAPI,
    ui: UIAPI,
    extension: ExtensionAPI,
    embed: EmbedAPI
}
```

---

## ProjectAPI

### `project.getProject()`

**Purpose:** Retrieve current project information

**Signature:**
```typescript
getProject(): Promise<Project>
```

**Returns:**
```typescript
interface Project {
    id: string;              // Project UUID
    name: string;            // Project name
    location: string;        // Region: 'us', 'europe', 'apac'
    address?: string;        // Project address
    crs?: string;           // Coordinate Reference System
    // Additional fields may exist
}
```

**Example:**
```javascript
const project = await API.project.getProject();
console.log('Project ID:', project.id);
console.log('Project Name:', project.name);
console.log('Region:', project.location);

// Use location to determine API endpoint
const apiHost = project.location === 'europe'
    ? 'app21.connect.trimble.com'
    : 'app.connect.trimble.com';
```

### `project.getCurrentProject()`

**Purpose:** Get active project context

**Signature:**
```typescript
getCurrentProject(): Promise<Project>
```

**Note:** Appears to be alias for `getProject()` in current implementation.

---

## UserAPI

### `user.getUserSettings()`

**Purpose:** Retrieve current user preferences

**Signature:**
```typescript
getUserSettings(): Promise<UserSettings>
```

**Returns:**
```typescript
interface UserSettings {
    language: string;        // User's language preference (e.g., 'en', 'de')
    // Additional settings fields
}
```

**Example:**
```javascript
const settings = await API.user.getUserSettings();
console.log('User language:', settings.language);

// Use for localization
if (settings.language === 'de') {
    loadGermanTranslations();
}
```

---

## UIAPI

### `ui.setMenu()`

**Purpose:** Register plugin menu item in Trimble Connect sidebar

**Signature:**
```typescript
setMenu(menuObject: MenuObject): Promise<void>
```

**MenuObject Interface:**
```typescript
interface MenuObject {
    title: string;           // Display name in sidebar
    icon?: string;           // Icon URL (SVG or PNG)
    command: string;         // Command ID to trigger
}
```

**Example:**
```javascript
const menuObject = {
    title: "My Plugin",
    icon: "https://yourdomain.github.io/plugin/icon.svg",
    command: "show_main_view"
};

await API.ui.setMenu(menuObject);

// When user clicks menu item, extension.command event fires:
// event = "extension.command"
// args.data = "show_main_view"
```

**Icon Requirements:**
- Format: SVG (recommended) or PNG
- Size: 24x24 to 128x128 pixels
- Color: Monochrome or color (sidebar may apply filters)
- Must be publicly accessible (HTTPS)

---

## ExtensionAPI

### `extension.getPermission()`

**Purpose:** Request permission from user

**Signature:**
```typescript
getPermission(permission: string): Promise<string | 'denied'>
```

**Supported Permissions:**
- `"accesstoken"`: Request OAuth access token

**Returns:**
- Access token string if granted
- `"denied"` if user rejects
- May also return token directly without user prompt if previously granted

**Example:**
```javascript
const token = await API.extension.getPermission("accesstoken");

if (token === 'denied') {
    console.log('User denied access token permission');
    showError('Permission required to access project data');
} else {
    console.log('Access token granted');
    storeToken(token); // Store in memory only
    loadProjectData(token);
}
```

**Critical Notes:**
- Method name is `getPermission` NOT `requestAccessToken`
- Token should be stored in memory, NOT localStorage
- Token may expire; listen for `extension.accessToken` event for updates

---

## EmbedAPI

### `embed.init3DViewer()`

**Purpose:** Embed Trimble's 3D model viewer

**Signature:**
```typescript
init3DViewer(options: ViewerOptions): Promise<ViewerInstance>
```

**Options:**
```typescript
interface ViewerOptions {
    projectId?: string;
    modelId?: string;
    versionId?: string;
    viewId?: string;
}
```

**Example:**
```javascript
const viewer = await API.embed.init3DViewer({
    projectId: project.id,
    modelId: 'model-uuid',
    versionId: 'version-uuid'
});

// Viewer embedded in your extension's UI
```

### `embed.initFileExplorer()`

**Purpose:** Embed Trimble's file explorer component

**Signature:**
```typescript
initFileExplorer(options: ExplorerOptions): Promise<ExplorerInstance>
```

**Options:**
```typescript
interface ExplorerOptions {
    projectId?: string;
    folderId?: string;
}
```

**Example:**
```javascript
const explorer = await API.embed.initFileExplorer({
    projectId: project.id
});

// File explorer embedded in your UI
// User can browse project files
```

**Limitation:** File explorer is UI-only; cannot extract file list programmatically.

### `embed.initProjectList()`

**Purpose:** Embed project list selector

**Signature:**
```typescript
initProjectList(options?: ProjectListOptions): Promise<ProjectListInstance>
```

**Example:**
```javascript
const projectList = await API.embed.initProjectList();
// User can select different projects
```

### `embed.setTokens()`

**Purpose:** Provide authentication tokens to embedded components

**Signature:**
```typescript
setTokens(tokens: TokenObject): void
```

**TokenObject:**
```typescript
interface TokenObject {
    accessToken: string;
}
```

**Example:**
```javascript
API.embed.setTokens({
    accessToken: myAccessToken
});

// Embedded components now authenticated
```

**Note:** Required for embedded components to access protected data.

---

## Events

Events are received through the callback function passed to `connect()`.

### Event: `extension.command`

**Trigger:** User clicks menu item
**Purpose:** Execute plugin action

**Data:**
```typescript
args.data: string  // Command ID from setMenu()
```

**Example:**
```javascript
function eventHandler(event, args) {
    if (event === "extension.command") {
        const commandId = args.data;

        switch (commandId) {
            case "show_main_view":
                displayMainView();
                break;
            case "show_settings":
                displaySettings();
                break;
            default:
                console.log('Unknown command:', commandId);
        }
    }
}
```

### Event: `extension.accessToken`

**Trigger:** Access token updated or refreshed
**Purpose:** Provide updated token to extension

**Data:**
```typescript
args.data: string  // New access token
```

**Example:**
```javascript
function eventHandler(event, args) {
    if (event === "extension.accessToken") {
        const newToken = args.data;
        console.log('Token updated');
        updateStoredToken(newToken);
    }
}
```

### Event: `extension.userSettingsChanged`

**Trigger:** User changes settings (language, etc.)
**Purpose:** Notify extension of settings change

**Example:**
```javascript
function eventHandler(event, args) {
    if (event === "extension.userSettingsChanged") {
        console.log('Settings changed, reloading...');
        reloadUserSettings();
    }
}
```

### Event: `extension.sessionInvalid`

**Trigger:** Session expired or authentication failed
**Purpose:** Notify extension to re-authenticate

**Example:**
```javascript
function eventHandler(event, args) {
    if (event === "extension.sessionInvalid") {
        console.log('Session invalid, requesting new token');
        requestNewToken();
    }
}
```

---

## Capabilities and Limitations

### ✅ Available via Workspace API

| Capability | Method | Notes |
|------------|--------|-------|
| Get project info | `project.getProject()` | Name, ID, region |
| Get user settings | `user.getUserSettings()` | Language, preferences |
| Register menu | `ui.setMenu()` | Sidebar integration |
| Request token | `extension.getPermission()` | OAuth access token |
| Embed 3D viewer | `embed.init3DViewer()` | UI component |
| Embed file explorer | `embed.initFileExplorer()` | UI component |
| Embed project list | `embed.initProjectList()` | UI component |

### ❌ NOT Available via Workspace API

| Capability | Reason | Solution |
|------------|--------|----------|
| List files programmatically | Not exposed | Use backend + REST API |
| Download file contents | Not exposed | Use backend + REST API |
| Upload files | Not exposed | Use backend + REST API |
| Query BIM models | Not exposed | Use backend + REST API |
| Access Topics/Issues | Not exposed | Use backend + REST API |
| Activity logs | Undocumented | Use backend or contact support |
| Create projects | Not exposed | Use backend + REST API |
| Manage users | Not exposed | Use backend + REST API |

---

## Complete Integration Example

```javascript
<!DOCTYPE html>
<html>
<head>
    <title>My Plugin</title>
</head>
<body>
    <div id="app">Loading...</div>

    <!-- Load Workspace API -->
    <script src="https://components.connect.trimble.com/trimble-connect-workspace-api/index.js"></script>

    <script>
        let globalAPI = null;
        let accessToken = null;

        async function init() {
            try {
                // Check if API loaded
                if (typeof TrimbleConnectWorkspace === 'undefined') {
                    throw new Error('Workspace API not loaded');
                }

                // Connect to API
                const API = await TrimbleConnectWorkspace.connect(
                    window.parent,
                    handleEvent,
                    30000
                );

                globalAPI = API;
                console.log('Connected to Trimble Connect');

                // Register menu
                await API.ui.setMenu({
                    title: "My Plugin",
                    icon: "https://example.com/icon.svg",
                    command: "show_main"
                });

                // Request access token
                const token = await API.extension.getPermission("accesstoken");
                if (token !== 'denied') {
                    accessToken = token;
                    await loadData();
                }

            } catch (error) {
                console.error('Init failed:', error);
                document.getElementById('app').textContent = 'Failed to connect';
            }
        }

        function handleEvent(event, args) {
            switch (event) {
                case "extension.command":
                    if (args.data === "show_main") {
                        displayMainView();
                    }
                    break;

                case "extension.accessToken":
                    accessToken = args.data;
                    loadData();
                    break;

                case "extension.userSettingsChanged":
                    reloadSettings();
                    break;
            }
        }

        async function loadData() {
            const project = await globalAPI.project.getProject();
            document.getElementById('app').innerHTML = `
                <h1>Project: ${project.name}</h1>
                <p>ID: ${project.id}</p>
                <p>Region: ${project.location}</p>
            `;
        }

        function displayMainView() {
            console.log('Showing main view');
            loadData();
        }

        function reloadSettings() {
            console.log('Settings changed');
        }

        // Start initialization
        init();
    </script>
</body>
</html>
```

---

## Debugging Tips

### Check API Load Status
```javascript
console.log('TrimbleConnectWorkspace loaded:', typeof TrimbleConnectWorkspace !== 'undefined');
```

### Log All Events
```javascript
function eventHandler(event, args) {
    console.log('Event:', event, 'Args:', args);
    // Helps discover undocumented events
}
```

### Test Outside Trimble Connect
```javascript
if (window.parent === window) {
    console.log('Running standalone (not in iframe)');
    // Mock API for local testing
} else {
    console.log('Running inside Trimble Connect');
}
```

---

## Next Steps

**For REST API access:** Read [03_REST_API.md](03_REST_API.md)
**For implementation:** Read [04_BASIC_PLUGIN.md](04_BASIC_PLUGIN.md)
