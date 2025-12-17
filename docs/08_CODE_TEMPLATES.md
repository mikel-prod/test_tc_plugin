# Code Templates

**AI Agent Instructions:** Copy-paste these tested code templates for common plugin patterns.

---

## Template 1: Minimal Plugin

**Use case:** Simplest possible plugin with project info display

**Files needed:** `index.html`, `manifest.json`

### manifest.json
```json
{
  "title": "Minimal Plugin",
  "url": "https://YOUR_USERNAME.github.io/REPO_NAME/index.html",
  "description": "Minimal Trimble Connect plugin",
  "enabled": true
}
```

### index.html
```html
<!DOCTYPE html>
<html>
<head>
    <title>Minimal Plugin</title>
    <style>
        body { font-family: sans-serif; padding: 20px; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background: #667eea; color: white; }
    </style>
</head>
<body>
    <h1>Project Info</h1>
    <div id="content">Loading...</div>

    <script src="https://components.connect.trimble.com/trimble-connect-workspace-api/index.js"></script>
    <script>
        (async () => {
            try {
                const API = await TrimbleConnectWorkspace.connect(window.parent, () => {}, 30000);

                await API.ui.setMenu({
                    title: "Minimal Plugin",
                    command: "show"
                });

                const project = await API.project.getProject();

                document.getElementById('content').innerHTML = `
                    <table>
                        <tr><th>Property</th><th>Value</th></tr>
                        <tr><td>Name</td><td>${project.name}</td></tr>
                        <tr><td>ID</td><td>${project.id}</td></tr>
                        <tr><td>Region</td><td>${project.location}</td></tr>
                    </table>
                `;
            } catch (error) {
                document.getElementById('content').textContent = 'Error: ' + error.message;
            }
        })();
    </script>
</body>
</html>
```

---

## Template 2: Event-Driven Plugin

**Use case:** Plugin that responds to menu clicks and events

### Complete Event Handler
```javascript
let globalAPI = null;
let accessToken = null;

async function init() {
    try {
        if (typeof TrimbleConnectWorkspace === 'undefined') {
            throw new Error('Workspace API not loaded');
        }

        const API = await TrimbleConnectWorkspace.connect(
            window.parent,
            handleEvent,
            30000
        );

        globalAPI = API;

        // Register menu
        await API.ui.setMenu({
            title: "My Plugin",
            icon: "https://example.com/icon.svg",
            command: "show_main"
        });

        // Request token
        const token = await API.extension.getPermission("accesstoken");
        if (token !== 'denied') {
            accessToken = token;
            onReady();
        }

    } catch (error) {
        console.error('Init failed:', error);
        showError(error.message);
    }
}

function handleEvent(event, args) {
    console.log('Event:', event, args);

    switch (event) {
        case "extension.command":
            handleCommand(args.data);
            break;

        case "extension.accessToken":
            accessToken = args.data;
            onTokenUpdated();
            break;

        case "extension.userSettingsChanged":
            onSettingsChanged();
            break;

        case "extension.sessionInvalid":
            onSessionInvalid();
            break;

        default:
            console.log('Unknown event:', event);
    }
}

function handleCommand(commandId) {
    console.log('Command:', commandId);

    switch (commandId) {
        case "show_main":
            showMainView();
            break;

        case "show_settings":
            showSettings();
            break;

        default:
            console.log('Unknown command:', commandId);
    }
}

function onReady() {
    console.log('Plugin ready');
    showMainView();
}

function onTokenUpdated() {
    console.log('Token updated');
    // Refresh data if needed
}

function onSettingsChanged() {
    console.log('Settings changed');
    // Reload settings
}

function onSessionInvalid() {
    console.log('Session invalid');
    // Re-authenticate
}

function showMainView() {
    // Implementation
}

function showSettings() {
    // Implementation
}

function showError(message) {
    document.getElementById('content').innerHTML =
        `<div class="error">${escapeHtml(message)}</div>`;
}

function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}

// Start
init();
```

---

## Template 3: Regional API Endpoint Handler

**Use case:** Making backend API calls with correct regional endpoint

### Get Regional API Host
```javascript
function getApiHost(projectLocation) {
    const hosts = {
        'europe': 'app21.connect.trimble.com',
        'eu': 'app21.connect.trimble.com',
        'apac': 'app31.connect.trimble.com',
        'asia': 'app31.connect.trimble.com',
        'us': 'app.connect.trimble.com'
    };
    return hosts[projectLocation?.toLowerCase()] || 'app.connect.trimble.com';
}

// Usage
async function callTrimbleAPI(endpoint) {
    const project = await globalAPI.project.getProject();
    const apiHost = getApiHost(project.location);
    const url = `https://${apiHost}${endpoint}`;

    const response = await fetch(url, {
        headers: {
            'Authorization': `Bearer ${accessToken}`,
            'Accept': 'application/json'
        }
    });

    if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
    }

    return response.json();
}
```

---

## Template 4: Backend Proxy Pattern

**Use case:** Full file access via backend server

### Frontend Code
```javascript
async function getProjectFiles() {
    try {
        const project = await globalAPI.project.getProject();

        // Call YOUR backend, not Trimble's API directly
        const response = await fetch('https://your-backend.vercel.app/api/files', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${accessToken}`
            },
            body: JSON.stringify({
                projectId: project.id,
                region: project.location
            })
        });

        if (!response.ok) {
            throw new Error(`Backend error: ${response.status}`);
        }

        const data = await response.json();
        return data.files;

    } catch (error) {
        console.error('Error fetching files:', error);
        throw error;
    }
}
```

### Backend Code (Vercel Serverless Function)
```javascript
// api/files.js

export default async function handler(req, res) {
    // Enable CORS
    res.setHeader('Access-Control-Allow-Origin', 'https://YOUR_USERNAME.github.io');
    res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS');
    res.setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type');

    if (req.method === 'OPTIONS') {
        return res.status(200).end();
    }

    if (req.method !== 'POST') {
        return res.status(405).json({ error: 'Method not allowed' });
    }

    try {
        // Get token from header
        const token = req.headers.authorization?.replace('Bearer ', '');
        if (!token) {
            return res.status(401).json({ error: 'No token provided' });
        }

        // Get parameters
        const { projectId, region } = req.body;
        if (!projectId) {
            return res.status(400).json({ error: 'projectId required' });
        }

        // Determine API host
        const apiHost = getApiHost(region);
        const apiUrl = `https://${apiHost}/tc/api/2.0/projects/${projectId}/files`;

        // Call Trimble API
        const response = await fetch(apiUrl, {
            headers: {
                'Authorization': `Bearer ${token}`,
                'Accept': 'application/json'
            }
        });

        if (!response.ok) {
            throw new Error(`Trimble API error: ${response.status}`);
        }

        const files = await response.json();

        // Return processed data
        return res.status(200).json({
            files: files,
            count: files.length,
            timestamp: new Date().toISOString()
        });

    } catch (error) {
        console.error('Error:', error);
        return res.status(500).json({ error: error.message });
    }
}

function getApiHost(region) {
    const hosts = {
        'europe': 'app21.connect.trimble.com',
        'eu': 'app21.connect.trimble.com',
        'apac': 'app31.connect.trimble.com',
        'asia': 'app31.connect.trimble.com',
        'us': 'app.connect.trimble.com'
    };
    return hosts[region?.toLowerCase()] || 'app.connect.trimble.com';
}
```

---

## Template 5: Error Handling Pattern

**Use case:** Robust error handling for all scenarios

```javascript
class PluginError extends Error {
    constructor(message, type, details = {}) {
        super(message);
        this.name = 'PluginError';
        this.type = type;
        this.details = details;
    }
}

async function safeAPICall(fn, errorMessage = 'Operation failed') {
    try {
        return await fn();
    } catch (error) {
        console.error(errorMessage, error);

        // Categorize error
        if (error.message?.includes('CORS')) {
            throw new PluginError(
                'Cannot access API directly from browser. Backend proxy required.',
                'CORS_ERROR',
                { originalError: error.message }
            );
        } else if (error.message?.includes('401') || error.message?.includes('403')) {
            throw new PluginError(
                'Authentication failed. Please refresh and try again.',
                'AUTH_ERROR',
                { originalError: error.message }
            );
        } else if (error.message?.includes('404')) {
            throw new PluginError(
                'Resource not found. Check project ID and region.',
                'NOT_FOUND',
                { originalError: error.message }
            );
        } else if (error.message?.includes('429')) {
            throw new PluginError(
                'Too many requests. Please wait and try again.',
                'RATE_LIMIT',
                { originalError: error.message }
            );
        } else {
            throw new PluginError(
                errorMessage + ': ' + error.message,
                'UNKNOWN_ERROR',
                { originalError: error.message }
            );
        }
    }
}

// Usage
async function loadData() {
    try {
        const project = await safeAPICall(
            () => globalAPI.project.getProject(),
            'Failed to get project'
        );

        const files = await safeAPICall(
            () => getProjectFiles(project.id),
            'Failed to get files'
        );

        displayFiles(files);

    } catch (error) {
        if (error instanceof PluginError) {
            showUserFriendlyError(error);
        } else {
            showGenericError(error);
        }
    }
}

function showUserFriendlyError(error) {
    const messages = {
        'CORS_ERROR': 'This feature requires a backend server. Please contact support.',
        'AUTH_ERROR': 'Please refresh the page and log in again.',
        'NOT_FOUND': 'Data not found. Please check your permissions.',
        'RATE_LIMIT': 'Too many requests. Please wait a moment and try again.',
        'UNKNOWN_ERROR': 'An unexpected error occurred. Please try again.'
    };

    const message = messages[error.type] || error.message;
    showError(message);
}
```

---

## Template 6: Loading States Pattern

**Use case:** Show loading indicators during async operations

```javascript
class LoadingManager {
    constructor(containerId) {
        this.container = document.getElementById(containerId);
    }

    showLoading(message = 'Loading...') {
        this.container.innerHTML = `
            <div class="loading">
                <div class="spinner"></div>
                <p>${escapeHtml(message)}</p>
            </div>
        `;
    }

    showError(message) {
        this.container.innerHTML = `
            <div class="error">
                <p>${escapeHtml(message)}</p>
            </div>
        `;
    }

    showContent(html) {
        this.container.innerHTML = html;
    }

    async execute(fn, loadingMessage) {
        this.showLoading(loadingMessage);
        try {
            const result = await fn();
            return result;
        } catch (error) {
            this.showError(error.message);
            throw error;
        }
    }
}

// CSS
const styles = `
.loading {
    text-align: center;
    padding: 40px;
}

.spinner {
    border: 4px solid #f3f3f3;
    border-top: 4px solid #667eea;
    border-radius: 50%;
    width: 40px;
    height: 40px;
    animation: spin 1s linear infinite;
    margin: 0 auto 20px;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

.error {
    background: #ffebee;
    color: #c62828;
    padding: 20px;
    border-radius: 10px;
    margin: 20px;
}
`;

// Usage
const loader = new LoadingManager('content');

async function loadProjectData() {
    await loader.execute(async () => {
        const project = await globalAPI.project.getProject();
        const html = `<h2>${project.name}</h2><p>ID: ${project.id}</p>`;
        loader.showContent(html);
    }, 'Loading project data...');
}
```

---

## Template 7: Local Development Mock

**Use case:** Test plugin outside Trimble Connect

```javascript
function createMockAPI() {
    return {
        project: {
            getProject: async () => ({
                id: 'mock-project-id',
                name: 'Mock Project',
                location: 'us',
                address: '123 Mock St',
                crs: 'WGS84'
            })
        },
        user: {
            getUserSettings: async () => ({
                language: 'en'
            })
        },
        ui: {
            setMenu: async (menuObj) => {
                console.log('[Mock] Menu set:', menuObj);
            }
        },
        extension: {
            getPermission: async (permission) => {
                console.log('[Mock] Permission requested:', permission);
                return 'mock-access-token';
            }
        }
    };
}

async function init() {
    let API;

    // Check if running inside Trimble Connect
    if (window.parent === window || typeof TrimbleConnectWorkspace === 'undefined') {
        console.log('Running in development mode with mock API');
        API = createMockAPI();

        // Add visual indicator
        document.body.insertAdjacentHTML('afterbegin', `
            <div style="background: #ffeb3b; padding: 10px; text-align: center; font-weight: bold;">
                DEVELOPMENT MODE - Using mock data
            </div>
        `);
    } else {
        console.log('Running in production mode');
        API = await TrimbleConnectWorkspace.connect(window.parent, handleEvent, 30000);
    }

    globalAPI = API;

    // Continue with normal initialization
    await API.ui.setMenu({ title: "My Plugin", command: "show" });
    const token = await API.extension.getPermission("accesstoken");
    // ...
}
```

---

## Template 8: File Statistics Processor

**Use case:** Process file lists to generate statistics

```javascript
function analyzeFiles(files) {
    const stats = {
        totalFiles: 0,
        totalFolders: 0,
        extensionCounts: {},
        largestFile: null,
        totalSize: 0
    };

    function processItem(item) {
        if (item.type === 'FILE') {
            stats.totalFiles++;

            // Count by extension
            const ext = item.name.split('.').pop().toLowerCase() || 'no-extension';
            stats.extensionCounts[ext] = (stats.extensionCounts[ext] || 0) + 1;

            // Track size
            if (item.size) {
                stats.totalSize += item.size;
                if (!stats.largestFile || item.size > stats.largestFile.size) {
                    stats.largestFile = item;
                }
            }
        } else if (item.type === 'FOLDER') {
            stats.totalFolders++;
        }
    }

    // Process flat array or nested tree
    if (Array.isArray(files)) {
        files.forEach(processItem);
    }

    // Sort extensions by count
    stats.sortedExtensions = Object.entries(stats.extensionCounts)
        .sort((a, b) => b[1] - a[1])
        .map(([ext, count]) => ({ extension: ext, count }));

    return stats;
}

function displayStatistics(stats) {
    const html = `
        <h2>File Statistics</h2>
        <table>
            <tr><td>Total Files</td><td>${stats.totalFiles}</td></tr>
            <tr><td>Total Folders</td><td>${stats.totalFolders}</td></tr>
            <tr><td>Total Size</td><td>${formatBytes(stats.totalSize)}</td></tr>
            <tr><td>Largest File</td><td>${stats.largestFile?.name || 'N/A'}</td></tr>
        </table>
        <h3>Files by Extension</h3>
        <table>
            <thead>
                <tr><th>Extension</th><th>Count</th></tr>
            </thead>
            <tbody>
                ${stats.sortedExtensions.map(item => `
                    <tr>
                        <td>.${item.extension}</td>
                        <td>${item.count}</td>
                    </tr>
                `).join('')}
            </tbody>
        </table>
    `;

    document.getElementById('content').innerHTML = html;
}

function formatBytes(bytes) {
    if (bytes === 0) return '0 Bytes';
    const k = 1024;
    const sizes = ['Bytes', 'KB', 'MB', 'GB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return Math.round(bytes / Math.pow(k, i) * 100) / 100 + ' ' + sizes[i];
}
```

---

## Template 9: Retry Logic with Exponential Backoff

**Use case:** Handle transient failures and rate limits

```javascript
async function retryWithBackoff(fn, maxRetries = 3, initialDelay = 1000) {
    let lastError;

    for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
            return await fn();
        } catch (error) {
            lastError = error;

            // Don't retry on certain errors
            if (error.message?.includes('401') ||
                error.message?.includes('403') ||
                error.message?.includes('404')) {
                throw error;
            }

            // Calculate delay with exponential backoff
            const delay = initialDelay * Math.pow(2, attempt);
            console.log(`Attempt ${attempt + 1} failed, retrying in ${delay}ms...`);

            // Wait before retry
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }

    throw new Error(`Failed after ${maxRetries} attempts: ${lastError.message}`);
}

// Usage
const files = await retryWithBackoff(
    () => fetchProjectFiles(projectId),
    3,    // max 3 retries
    1000  // start with 1 second delay
);
```

---

## Template 10: GitHub Actions Workflow

**Use case:** Automated deployment to GitHub Pages

**File:** `.github/workflows/deploy.yml`

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: '.'

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

---

## Quick Reference: Common Operations

### Get Project Info
```javascript
const project = await globalAPI.project.getProject();
```

### Get User Settings
```javascript
const settings = await globalAPI.user.getUserSettings();
```

### Register Menu
```javascript
await globalAPI.ui.setMenu({
    title: "My Plugin",
    command: "show"
});
```

### Get Access Token
```javascript
const token = await globalAPI.extension.getPermission("accesstoken");
```

### Escape HTML
```javascript
function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}
```

### Format Date
```javascript
function formatDate(dateString) {
    return new Date(dateString).toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
    });
}
```

---

## Next Steps

**Implement your plugin:** Combine these templates as needed
**Debug issues:** See [07_QUIRKS_AND_GOTCHAS.md](07_QUIRKS_AND_GOTCHAS.md)
**Deploy:** See [06_DEPLOYMENT.md](06_DEPLOYMENT.md)
