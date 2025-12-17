# Trimble Connect Plugin Overview

**AI Agent Instructions:** Read this document first to understand the platform architecture before implementation.

---

## What is Trimble Connect?

**Type:** Cloud-based construction project collaboration platform
**Purpose:** File sharing, model viewing, issue tracking, project management
**Architecture:** Web + Desktop + Mobile applications
**Regions:** US, Europe (EU), Asia-Pacific (APAC) - each with separate API hosts

---

## Plugin Types

### Type 1: Browser Extension (Workspace API)
**Location:** Runs inside Trimble Connect web application
**Technology:** HTML + JavaScript in iframe
**Hosting:** Static hosting (GitHub Pages, Netlify, etc.)
**Access:** Limited to Workspace API capabilities

**Architecture:**
```
┌─────────────────────────────────┐
│ Trimble Connect Web Application │
│ ┌─────────────────────────────┐ │
│ │ Your Plugin (iframe)        │ │
│ │ - HTML/CSS/JavaScript       │ │
│ │ - Workspace API access      │ │
│ └─────────────────────────────┘ │
└─────────────────────────────────┘
```

### Type 2: Backend-Integrated Extension
**Location:** Frontend in Trimble Connect + Backend server
**Technology:** Frontend (HTML/JS) + Backend (Node.js/Python/.NET)
**Hosting:** Frontend (static) + Backend (Vercel/AWS/Azure)
**Access:** Full REST API capabilities via backend proxy

**Architecture:**
```
┌─────────────────────────────────┐
│ Trimble Connect Web Application │
│ ┌─────────────────────────────┐ │
│ │ Your Plugin (iframe)        │ │
│ │ Frontend: HTML/JS           │ │
│ └──────────┬──────────────────┘ │
└────────────┼────────────────────┘
             │ HTTPS
             ▼
┌─────────────────────────────────┐
│ Your Backend Server             │
│ - API Proxy                     │
│ - Business Logic                │
│ - Database (optional)           │
└──────────┬──────────────────────┘
           │ HTTPS + Bearer Token
           ▼
┌─────────────────────────────────┐
│ Trimble Connect REST APIs       │
│ - Core API                      │
│ - Model API                     │
│ - Topics API                    │
└─────────────────────────────────┘
```

---

## Key Platform Components

### 1. Workspace API
**Purpose:** Browser-based integration API
**Access Method:** Global JavaScript object `TrimbleConnectWorkspace`
**Loaded From:** CDN script tag

**Capabilities:**
- ✅ Get project metadata (name, ID, region)
- ✅ Get user settings (language, preferences)
- ✅ Register sidebar menu items
- ✅ Embed viewers (3D, file explorer, project list)
- ✅ Request access tokens
- ❌ List files programmatically
- ❌ Download/upload files
- ❌ Query BIM models
- ❌ Access activity logs

### 2. REST APIs
**Purpose:** Server-to-server data access
**Access Method:** HTTPS requests with Bearer token
**Base URLs:** Regional (see Regional Architecture below)

**Available APIs:**
- **Core API:** Files, folders, projects, users
- **Model API:** BIM model queries
- **Organizer API:** Hierarchical structures
- **Property Set API:** Custom properties
- **Topics API:** BCF issues and comments
- **Model Feature API:** Read Organizer groups

### 3. Authentication
**Method:** OAuth 2.0 Authorization Code Flow with PKCE
**Token Types:** Access token (short-lived), Refresh token (9-day max)
**User Context:** Tokens tied to authenticated user
**Service Accounts:** NOT supported (requires user interaction)

---

## Regional Architecture

**Critical:** API endpoints vary by project region.

| Region | Web UI Host | API Host |
|--------|-------------|----------|
| US (default) | web.connect.trimble.com | app.connect.trimble.com |
| Europe | web21.connect.trimble.com | app21.connect.trimble.com |
| Asia-Pacific | web31.connect.trimble.com | app31.connect.trimble.com |

**Implementation Pattern:**
```javascript
function getApiHost(projectLocation) {
    const hosts = {
        'europe': 'app21.connect.trimble.com',
        'eu': 'app21.connect.trimble.com',
        'apac': 'app31.connect.trimble.com',
        'asia': 'app31.connect.trimble.com',
        'us': 'app.connect.trimble.com'
    };
    return hosts[projectLocation] || 'app.connect.trimble.com';
}

// Get project location from Workspace API
const project = await API.project.getProject();
const apiHost = getApiHost(project.location);
```

---

## Plugin Lifecycle

### 1. Installation
User adds manifest URL to their Trimble Connect project:
- Project Settings → Extensions → Add Extension
- Enters manifest URL (e.g., `https://yourdomain.github.io/plugin/manifest.json`)
- Extension appears in sidebar

### 2. Loading
1. Trimble Connect loads your HTML in iframe
2. Your script loads Workspace API from CDN
3. Call `TrimbleConnectWorkspace.connect()` to establish connection
4. Register menu items via `API.ui.setMenu()`
5. Request permissions if needed

### 3. Execution
1. User clicks menu item
2. Extension receives `extension.command` event
3. Execute plugin logic
4. Display results in UI

### 4. Token Management
1. Call `API.extension.getPermission("accesstoken")` on init
2. User grants permission (one-time)
3. Store token in memory (never localStorage)
4. Use token for API calls
5. Listen for `extension.accessToken` event for updates

---

## Data Access Patterns

### Pattern 1: Workspace API Only (No Backend)
```javascript
// ✅ Works: Get project info
const project = await API.project.getProject();

// ✅ Works: Get user settings
const settings = await API.user.getUserSettings();

// ❌ Blocked: Direct REST API call (CORS)
const response = await fetch('https://app.connect.trimble.com/tc/api/2.0/projects/ID/files', {
    headers: { 'Authorization': `Bearer ${token}` }
});
// Error: CORS policy blocks this request
```

### Pattern 2: Backend Proxy (Full Access)
```javascript
// Frontend: Get token from Workspace API
const token = await API.extension.getPermission("accesstoken");
const project = await API.project.getProject();

// Frontend: Call YOUR backend
const response = await fetch('https://your-backend.vercel.app/api/files', {
    headers: {
        'Authorization': `Bearer ${token}`,
        'X-Project-ID': project.id,
        'X-Region': project.location
    }
});

// Backend: Proxy to Trimble API (no CORS issues)
const apiHost = getApiHost(headers['x-region']);
const trimbleResponse = await fetch(
    `https://${apiHost}/tc/api/2.0/projects/${projectId}/files`,
    {
        headers: { 'Authorization': req.headers.authorization }
    }
);
```

---

## File Structure

### Minimal Plugin (Static)
```
plugin/
├── manifest.json       # Extension configuration
├── index.html          # Plugin UI
└── icon.svg           # Extension icon (optional)
```

### Advanced Plugin (With Backend)
```
plugin/
├── frontend/
│   ├── manifest.json
│   ├── index.html
│   ├── app.js
│   └── icon.svg
└── backend/
    ├── api/
    │   └── proxy.js    # Serverless function
    ├── package.json
    └── vercel.json     # Deployment config
```

---

## Manifest File Format

**Critical:** Must be publicly accessible with CORS enabled.

**Minimal Example:**
```json
{
  "title": "My Plugin",
  "url": "https://yourdomain.github.io/plugin/index.html",
  "description": "Plugin description",
  "enabled": true
}
```

**Optional Fields:**
```json
{
  "title": "My Plugin",
  "url": "https://yourdomain.github.io/plugin/index.html",
  "description": "Plugin description",
  "icon": "https://yourdomain.github.io/plugin/icon.svg",
  "infoUrl": "https://github.com/yourname/plugin",
  "configCommand": "show_config",
  "enabled": true
}
```

**Field Descriptions:**
- `title`: Display name in sidebar (required)
- `url`: Plugin HTML URL (required, must be HTTPS)
- `description`: Short description (required)
- `icon`: Icon URL (optional, SVG or PNG)
- `infoUrl`: Documentation URL (optional)
- `configCommand`: Command ID for settings (optional)
- `enabled`: Enable/disable (required)

---

## Security Considerations

### ✅ Do
- Store access tokens in memory only
- Validate all user inputs
- Use HTTPS for all resources
- Implement CORS properly on backend
- Rate limit API calls
- Log errors (without sensitive data)

### ❌ Don't
- Store tokens in localStorage/sessionStorage
- Log tokens to console in production
- Trust iframe postMessage without validation
- Expose API keys in frontend code
- Use wildcards in CORS configuration
- Make unauthenticated API calls

---

## Performance Considerations

### Optimization Strategies
1. **Lazy load resources:** Don't load everything on init
2. **Cache API responses:** Store in memory for session
3. **Debounce user inputs:** Avoid excessive API calls
4. **Use pagination:** Don't fetch all data at once
5. **Minimize iframe size:** Smaller = faster loading

### API Rate Limits
- **Documented:** Not explicitly stated in public docs
- **Observed:** Rate limiting exists (implement backoff)
- **Best Practice:** Cache responses, batch requests

---

## Browser Compatibility

**Supported Browsers:**
- Chrome/Edge (Chromium) ✅
- Firefox ✅
- Safari ⚠️ (limited testing)

**Required Features:**
- ES6+ JavaScript
- Fetch API
- Promises/async-await
- PostMessage API (for iframe communication)

---

## Next Steps

**For Basic Plugin:** Read [04_BASIC_PLUGIN.md](04_BASIC_PLUGIN.md)
**For Advanced Plugin:** Read [02_WORKSPACE_API.md](02_WORKSPACE_API.md) → [03_REST_API.md](03_REST_API.md) → [05_ADVANCED_PLUGIN.md](05_ADVANCED_PLUGIN.md)
