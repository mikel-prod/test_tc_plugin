# Basic Plugin Implementation Guide

**AI Agent Instructions:** Follow these steps to create a working basic Trimble Connect plugin from scratch.

---

## Prerequisites

- GitHub account
- Text editor or IDE
- Git installed locally
- Basic knowledge of HTML/JavaScript

---

## Step-by-Step Implementation

### Step 1: Create Project Structure

```bash
mkdir my-tc-plugin
cd my-tc-plugin

# Create files
touch index.html
touch manifest.json
touch icon.svg
touch .gitignore
touch README.md
```

**Directory structure:**
```
my-tc-plugin/
├── index.html
├── manifest.json
├── icon.svg
├── .gitignore
└── README.md
```

---

### Step 2: Create .gitignore

**File:** `.gitignore`

```
# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log

# Build
node_modules/
dist/
```

---

### Step 3: Create Manifest

**File:** `manifest.json`

```json
{
  "title": "My Plugin",
  "url": "https://YOUR_GITHUB_USERNAME.github.io/my-tc-plugin/index.html",
  "description": "My Trimble Connect plugin",
  "enabled": true
}
```

**Critical:** Replace `YOUR_GITHUB_USERNAME` with your actual GitHub username.

**Field explanations:**
- `title`: Display name in Trimble Connect sidebar (30 chars max recommended)
- `url`: Full HTTPS URL to your index.html (MUST be HTTPS)
- `description`: Short description (100 chars max recommended)
- `enabled`: Must be `true` for plugin to load

---

### Step 4: Create Icon

**File:** `icon.svg`

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="128" height="128" viewBox="0 0 128 128">
  <defs>
    <linearGradient id="grad" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#667eea;stop-opacity:1" />
      <stop offset="100%" style="stop-color:#764ba2;stop-opacity:1" />
    </linearGradient>
  </defs>
  <rect width="128" height="128" rx="20" fill="url(#grad)"/>
  <text x="64" y="85" font-family="Arial, sans-serif" font-size="60" font-weight="bold" fill="white" text-anchor="middle">MP</text>
</svg>
```

**Replace "MP"** with your plugin initials.

**Icon requirements:**
- Format: SVG (recommended) or PNG
- Size: 128x128 pixels minimum
- Keep simple and recognizable at small sizes

---

### Step 5: Create Plugin HTML

**File:** `index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Trimble Connect Plugin</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            padding: 20px;
        }

        .container {
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
            padding: 40px;
            max-width: 600px;
            width: 100%;
        }

        h1 {
            font-size: 2rem;
            color: #333;
            margin-bottom: 20px;
        }

        .info {
            margin-top: 20px;
            padding: 15px;
            background: #f8f9fa;
            border-radius: 10px;
            font-size: 0.9rem;
        }

        .info strong {
            color: #333;
        }

        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }

        th, td {
            padding: 10px;
            text-align: left;
            border-bottom: 1px solid #eee;
        }

        th {
            background: #667eea;
            color: white;
        }

        .error {
            background: #ffebee;
            color: #c62828;
            padding: 15px;
            border-radius: 10px;
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>My Plugin</h1>
        <div id="content">
            <div class="info">Loading...</div>
        </div>
    </div>

    <!-- Load Trimble Connect Workspace API -->
    <script src="https://components.connect.trimble.com/trimble-connect-workspace-api/index.js"></script>

    <script>
        let globalAPI = null;
        let accessToken = null;

        // Initialize plugin
        async function init() {
            try {
                // Check if Workspace API loaded
                console.log('Checking Workspace API...');
                console.log('TrimbleConnectWorkspace available:', typeof TrimbleConnectWorkspace);

                if (typeof TrimbleConnectWorkspace === 'undefined') {
                    throw new Error('Workspace API not loaded');
                }

                // Connect to Trimble Connect
                console.log('Connecting to Trimble Connect...');
                const API = await TrimbleConnectWorkspace.connect(
                    window.parent,
                    handleEvent,
                    30000
                );

                globalAPI = API;
                console.log('Connected successfully!');

                // Register menu item in sidebar
                const menuObject = {
                    title: "My Plugin",
                    icon: "https://YOUR_GITHUB_USERNAME.github.io/my-tc-plugin/icon.svg",
                    command: "show_main_view"
                };

                await API.ui.setMenu(menuObject);
                console.log('Menu registered');

                // Request access token
                console.log('Requesting access token...');
                const token = await API.extension.getPermission("accesstoken");
                console.log('Token result:', token === 'denied' ? 'denied' : 'granted');

                if (token && token !== 'denied') {
                    accessToken = token;
                    await loadProjectInfo();
                } else {
                    showError('Access token required. Please grant permission.');
                }

            } catch (error) {
                console.error('Initialization failed:', error);
                showError('Failed to initialize: ' + error.message);
            }
        }

        // Handle events from Trimble Connect
        function handleEvent(event, args) {
            console.log('Event received:', event, args);

            switch (event) {
                case "extension.command":
                    handleCommand(args.data);
                    break;

                case "extension.accessToken":
                    console.log('Token updated');
                    accessToken = args.data;
                    if (accessToken) {
                        loadProjectInfo();
                    }
                    break;

                case "extension.userSettingsChanged":
                    console.log('User settings changed');
                    break;

                default:
                    console.log('Unknown event:', event);
            }
        }

        // Handle menu commands
        function handleCommand(commandId) {
            console.log('Command received:', commandId);

            switch (commandId) {
                case "show_main_view":
                    console.log('Showing main view');
                    if (accessToken) {
                        loadProjectInfo();
                    }
                    break;

                default:
                    console.log('Unknown command:', commandId);
            }
        }

        // Load and display project information
        async function loadProjectInfo() {
            try {
                console.log('Loading project info...');

                // Get project data from Workspace API
                const project = await globalAPI.project.getProject();
                console.log('Project data:', project);

                // Get user settings
                const userSettings = await globalAPI.user.getUserSettings();
                console.log('User settings:', userSettings);

                // Display data
                displayInfo(project, userSettings);

            } catch (error) {
                console.error('Error loading project info:', error);
                showError('Failed to load data: ' + error.message);
            }
        }

        // Display project information in UI
        function displayInfo(project, userSettings) {
            const html = `
                <table>
                    <thead>
                        <tr>
                            <th>Property</th>
                            <th>Value</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr>
                            <td>Project Name</td>
                            <td>${escapeHtml(project.name || 'N/A')}</td>
                        </tr>
                        <tr>
                            <td>Project ID</td>
                            <td>${escapeHtml(project.id || 'N/A')}</td>
                        </tr>
                        <tr>
                            <td>Region</td>
                            <td>${escapeHtml(project.location || 'N/A')}</td>
                        </tr>
                        <tr>
                            <td>Address</td>
                            <td>${escapeHtml(project.address || 'N/A')}</td>
                        </tr>
                        <tr>
                            <td>User Language</td>
                            <td>${escapeHtml(userSettings.language || 'N/A')}</td>
                        </tr>
                    </tbody>
                </table>
                <div class="info" style="margin-top: 20px;">
                    <strong>Note:</strong> This is a basic plugin showing data from the Workspace API.
                    For file access and advanced features, a backend server is required.
                </div>
            `;

            document.getElementById('content').innerHTML = html;
        }

        // Show error message
        function showError(message) {
            const html = `<div class="error">${escapeHtml(message)}</div>`;
            document.getElementById('content').innerHTML = html;
        }

        // Escape HTML to prevent XSS
        function escapeHtml(text) {
            const div = document.createElement('div');
            div.textContent = text;
            return div.innerHTML;
        }

        // Start initialization when page loads
        init();
    </script>
</body>
</html>
```

**Critical:** Replace `YOUR_GITHUB_USERNAME` in the icon URL.

---

### Step 6: Create README

**File:** `README.md`

```markdown
# My Trimble Connect Plugin

A simple Trimble Connect plugin that displays project information.

## Features

- Displays current project name, ID, and region
- Shows user language preference
- Clean, modern UI

## Installation

1. Go to your Trimble Connect project
2. Open Project Settings → Extensions
3. Add extension with this manifest URL:
   ```
   https://YOUR_GITHUB_USERNAME.github.io/my-tc-plugin/manifest.json
   ```

## Development

This plugin uses:
- Trimble Connect Workspace API
- Static HTML/JavaScript
- Hosted on GitHub Pages

## License

MIT
```

---

### Step 7: Initialize Git Repository

```bash
git init
git add .
git commit -m "Initial commit: Basic Trimble Connect plugin"
```

---

### Step 8: Create GitHub Repository

1. Go to [GitHub](https://github.com) and sign in
2. Click the **+** icon → **New repository**
3. Repository name: `my-tc-plugin`
4. Visibility: **Public** (required for free GitHub Pages)
5. **Do NOT** initialize with README (you already have one)
6. Click **Create repository**

---

### Step 9: Push to GitHub

```bash
# Add remote (replace YOUR_GITHUB_USERNAME)
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/my-tc-plugin.git

# Rename branch to main (if needed)
git branch -M main

# Push to GitHub
git push -u origin main
```

---

### Step 10: Enable GitHub Pages

1. Go to your repository on GitHub
2. Click **Settings** → **Pages** (in left sidebar)
3. Under **Source**:
   - Select: **Deploy from a branch**
   - Branch: **main**
   - Folder: **/ (root)**
4. Click **Save**

Wait 1-2 minutes for deployment.

---

### Step 11: Verify Deployment

Open these URLs in browser:

1. **Manifest:**
   ```
   https://YOUR_GITHUB_USERNAME.github.io/my-tc-plugin/manifest.json
   ```
   Should show JSON content

2. **Plugin page:**
   ```
   https://YOUR_GITHUB_USERNAME.github.io/my-tc-plugin/index.html
   ```
   Should show plugin UI (may not function fully outside Trimble Connect)

3. **Icon:**
   ```
   https://YOUR_GITHUB_USERNAME.github.io/my-tc-plugin/icon.svg
   ```
   Should display icon

If any URL returns 404, wait a few more minutes and try again.

---

### Step 12: Install in Trimble Connect

1. Log in to [Trimble Connect](https://connect.trimble.com/)
2. Open a project
3. Go to **Project Settings** (gear icon) → **Extensions**
4. In the **Extension manifest URL** field, enter:
   ```
   https://YOUR_GITHUB_USERNAME.github.io/my-tc-plugin/manifest.json
   ```
5. Click **Add** or **Install**
6. Grant permissions when prompted
7. Look for "My Plugin" in the left sidebar
8. Click it to open

---

## Testing Your Plugin

### In Trimble Connect

1. **Check sidebar:** Should see "My Plugin" menu item
2. **Click menu item:** Should display project information table
3. **Check browser console:**
   - Press F12 to open DevTools
   - Look for log messages
   - Check for any errors (red text)

### Common Issues

**Problem:** Plugin doesn't appear in sidebar

**Solutions:**
- Wait 5 minutes for GitHub Pages deployment
- Hard refresh browser (Ctrl+Shift+R)
- Verify manifest URL is correct
- Check GitHub Actions tab for deployment status
- Ensure repository is public

**Problem:** "Workspace API not loaded" error

**Solutions:**
- Check internet connection
- Verify script tag in HTML is correct
- Try refreshing page

**Problem:** "Failed to connect" error

**Solutions:**
- Ensure running inside Trimble Connect (not standalone)
- Check browser console for specific error
- Verify you're using latest browsers (Chrome/Edge/Firefox)

---

## Making Changes

After modifying files:

```bash
# Stage changes
git add .

# Commit
git commit -m "Description of changes"

# Push to GitHub
git push
```

Wait 1-2 minutes for GitHub Pages to update, then hard refresh in browser (Ctrl+Shift+R).

---

## Customization Ideas

### Change Colors
Edit gradient in `index.html`:
```css
background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
```

### Add More Data
Modify `displayInfo()` function to show additional project properties.

### Add Multiple Menu Items
Create multiple command IDs and handle them in `handleCommand()`.

### Style Improvements
Add more CSS to match your branding.

---

## Limitations of Basic Plugin

This basic plugin can:
- ✅ Display project metadata
- ✅ Display user settings
- ✅ Show custom UI

This basic plugin CANNOT:
- ❌ List project files
- ❌ Download/upload files
- ❌ Query BIM models
- ❌ Access activity logs

**For advanced features:** Read [05_ADVANCED_PLUGIN.md](../docs/05_ADVANCED_PLUGIN.md)

---

## Debugging Tips

### Enable Verbose Logging
```javascript
// Add at top of script
const DEBUG = true;

function log(...args) {
    if (DEBUG) {
        console.log('[My Plugin]', ...args);
    }
}

// Use throughout code
log('Initializing...');
```

### Check API Connection
```javascript
console.log('Window parent:', window.parent !== window);
console.log('API available:', typeof TrimbleConnectWorkspace !== 'undefined');
console.log('API connected:', !!globalAPI);
console.log('Token present:', !!accessToken);
```

### Test Locally with Mock API
```javascript
if (window.parent === window) {
    // Running standalone - create mock API
    window.TrimbleConnectWorkspace = {
        connect: async () => ({
            project: {
                getProject: async () => ({
                    id: 'test-id',
                    name: 'Test Project',
                    location: 'us'
                })
            },
            user: {
                getUserSettings: async () => ({
                    language: 'en'
                })
            },
            ui: {
                setMenu: async () => {}
            },
            extension: {
                getPermission: async () => 'mock-token'
            }
        })
    };
}
```

---

## Next Steps

- **Add backend:** [05_ADVANCED_PLUGIN.md](05_ADVANCED_PLUGIN.md)
- **Learn API details:** [02_WORKSPACE_API.md](02_WORKSPACE_API.md)
- **Avoid common issues:** [07_QUIRKS_AND_GOTCHAS.md](07_QUIRKS_AND_GOTCHAS.md)
- **Use code templates:** [08_CODE_TEMPLATES.md](08_CODE_TEMPLATES.md)

---

## Complete File Checklist

Before pushing to GitHub, verify:

- [ ] `.gitignore` created
- [ ] `manifest.json` created with YOUR GitHub username
- [ ] `index.html` created with YOUR GitHub username in icon URL
- [ ] `icon.svg` created
- [ ] `README.md` created
- [ ] All files committed to git
- [ ] Pushed to GitHub
- [ ] GitHub Pages enabled
- [ ] Repository is public
- [ ] Manifest URL accessible in browser
- [ ] Plugin tested in Trimble Connect

---

**Success Criteria:**

✅ Plugin appears in Trimble Connect sidebar
✅ Clicking plugin shows project information
✅ No errors in browser console
✅ Deployment automated via GitHub Pages

**You now have a working Trimble Connect plugin!**
