# Quirks and Gotchas

**AI Agent Instructions:** This document contains critical issues, workarounds, and non-obvious behaviors. Read carefully before implementing.

---

## Critical Errors and Fixes

### 1. Wrong Global Variable Name

**Error:**
```javascript
if (typeof WorkspaceAPI !== 'undefined') {
    // This will NEVER be true
}
```

**Reason:** The global variable is `TrimbleConnectWorkspace`, NOT `WorkspaceAPI`

**Fix:**
```javascript
if (typeof TrimbleConnectWorkspace !== 'undefined') {
    // ✅ Correct
}
```

**Symptoms:**
- "WorkspaceAPI is not defined"
- Extension fails to connect
- Blank screen with no errors (if not checking existence)

---

### 2. Wrong Access Token Method

**Error:**
```javascript
const token = await API.extension.requestAccessToken();
// TypeError: API.extension.requestAccessToken is not a function
```

**Reason:** Method name is `getPermission()`, not `requestAccessToken()`

**Fix:**
```javascript
const token = await API.extension.getPermission("accesstoken");
```

**Symptoms:**
- "requestAccessToken is not a function"
- Cannot get access token
- Permission dialog never shows

---

### 3. CORS Blocking Direct API Calls

**Error:**
```javascript
const response = await fetch('https://app.connect.trimble.com/tc/api/2.0/projects/ID/files', {
    headers: { 'Authorization': `Bearer ${token}` }
});
// CORS policy error
```

**Reason:** Browser blocks cross-origin requests from GitHub Pages (or any hosting) to Trimble Connect API

**Symptoms:**
- "Access to fetch... has been blocked by CORS policy"
- "No 'Access-Control-Allow-Origin' header"
- Network request shows as "failed" or "CORS error"

**Fix:** Use backend proxy server

**Workaround (if backend impossible):**
- Use only Workspace API methods
- Accept limitations (no file listing, etc.)
- Embed Trimble's components instead

---

### 4. Wrong Regional Endpoint

**Error:**
```javascript
const url = 'https://app.connect.trimble.com/tc/api/2.0/projects/ID/files';
// 404 Not Found (for European projects)
```

**Reason:** European and APAC projects use different API hosts

**Symptoms:**
- 404 Not Found errors
- CORS errors
- "Project not found"

**Fix:** Map project location to correct host

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

const project = await API.project.getProject();
const apiHost = getApiHost(project.location);
const url = `https://${apiHost}/tc/api/2.0/projects/${project.id}/files`;
```

---

### 5. Manifest Not Loading

**Error:** Extension doesn't appear in Trimble Connect after adding manifest URL

**Common Causes:**

**A. CORS not enabled on hosting**
```
Manifest URL: https://example.com/manifest.json
Error: CORS policy blocks manifest loading
```

**Fix:** Enable CORS on your server
- GitHub Pages: Automatic ✅
- Netlify: Automatic ✅
- Custom server: Add headers:
  ```
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: GET
  ```

**B. HTTPS required**
```
Manifest URL: http://example.com/manifest.json
Error: Trimble Connect requires HTTPS
```

**Fix:** Use HTTPS only (GitHub Pages provides this automatically)

**C. Invalid JSON**
```json
{
  "title": "My Plugin",
  "url": "https://example.com/index.html",
  "description": "Description"
  // Missing: "enabled": true
}
```

**Fix:** Validate JSON and ensure required fields

**D. Private repository**
```
GitHub repo: Private
Manifest URL: https://username.github.io/repo/manifest.json
Error: 404 Not Found
```

**Fix:** Make repository public (required for GitHub Pages free tier)

---

## Non-Obvious Behaviors

### 6. Access Token Can Be Granted Without Prompt

**Behavior:**
```javascript
const token = await API.extension.getPermission("accesstoken");
// Sometimes shows permission dialog
// Sometimes returns token immediately
```

**Reason:** If user previously granted permission, no prompt shown

**Implication:** Don't rely on dialog appearance for UX flow

**Pattern:**
```javascript
const token = await API.extension.getPermission("accesstoken");
if (token === 'denied') {
    // User explicitly denied
    showPermissionDeniedMessage();
} else {
    // Token received (with or without prompt)
    useToken(token);
}
```

---

### 7. Event Handler Called Multiple Times

**Behavior:**
```javascript
function eventHandler(event, args) {
    console.log('Event:', event);
    // May be called multiple times for same event
}
```

**Reason:** Trimble Connect may emit duplicate events in certain scenarios

**Fix:** Implement idempotent event handling

```javascript
let isProcessing = false;

function eventHandler(event, args) {
    if (event === "extension.command" && !isProcessing) {
        isProcessing = true;
        try {
            handleCommand(args.data);
        } finally {
            isProcessing = false;
        }
    }
}
```

---

### 8. Token Storage in localStorage Persists Across Sessions

**Behavior:**
```javascript
// ❌ DON'T DO THIS
localStorage.setItem('token', accessToken);

// Token still there after browser restart
// Token may be expired
// Security risk if device shared
```

**Reason:** localStorage persists indefinitely

**Security Issue:** Tokens in localStorage accessible to any script on same origin

**Fix:** Store in memory only

```javascript
// ✅ Store in memory
let accessToken = null;

// ✅ Listen for token updates
function eventHandler(event, args) {
    if (event === "extension.accessToken") {
        accessToken = args.data;
    }
}
```

---

### 9. Iframe Restrictions

**Behavior:**
```javascript
// Some browser APIs restricted in iframe
window.open('https://example.com'); // May be blocked
navigator.geolocation.getCurrentPosition(); // May not work
window.alert('Hello'); // May appear behind Trimble Connect UI
```

**Reason:** Extensions run in sandboxed iframe

**Limitations:**
- Pop-ups may be blocked
- Some browser APIs unavailable
- CSS may be affected by parent page styles

**Workarounds:**
- Use postMessage to communicate with parent
- Open links with `target="_blank"` attribute
- Test thoroughly in actual Trimble Connect environment

---

### 10. GitHub Pages Deployment Delay

**Behavior:**
```
Git push → GitHub Actions → Pages Deploy
Expected time: 1-2 minutes
Actual time: Sometimes 5-10 minutes
```

**Reason:** GitHub Actions queue and Pages CDN propagation

**Implication:** Changes not immediately visible

**Workflow:**
1. Push changes
2. Wait for GitHub Actions (check Actions tab)
3. Wait for Pages deployment
4. Hard refresh in browser (Ctrl+Shift+R)
5. If still showing old version, clear browser cache

---

### 11. Browser Cache Showing Old Version

**Behavior:**
```
- Updated plugin on GitHub Pages
- GitHub Actions shows successful deployment
- Browser still shows old version
```

**Reason:** Browser caches HTML/JS/CSS aggressively

**Fix:**
1. Hard refresh: Ctrl+Shift+R (Windows/Linux) or Cmd+Shift+R (Mac)
2. Clear cache for specific site
3. Open in incognito/private window
4. Add cache-busting to script URLs:
   ```html
   <script src="app.js?v=1.0.1"></script>
   ```

---

### 12. Manifest Changes Require Extension Reload

**Behavior:**
```
- Changed manifest.json title/icon
- Refreshed browser
- Sidebar still shows old title/icon
```

**Reason:** Trimble Connect caches manifest

**Fix:**
1. Remove extension from project settings
2. Re-add extension with manifest URL
3. Or: Wait for cache expiration (varies)

---

## API Endpoint Quirks

### 13. Files API Returns Flat List

**Behavior:**
```javascript
GET /tc/api/2.0/projects/{projectId}/files
// Returns array of files, not folder hierarchy
```

**Expected:** Nested folder structure
**Actual:** Flat array with `parentId` references

**Implication:** Must reconstruct folder tree manually

**Pattern:**
```javascript
const files = await fetchFiles(projectId);

function buildFolderTree(files) {
    const tree = {};
    const itemsById = {};

    // Index all items
    files.forEach(item => {
        itemsById[item.id] = { ...item, children: [] };
    });

    // Build tree
    files.forEach(item => {
        if (item.parentId) {
            const parent = itemsById[item.parentId];
            if (parent) {
                parent.children.push(itemsById[item.id]);
            }
        } else {
            tree[item.id] = itemsById[item.id];
        }
    });

    return tree;
}
```

---

### 14. Activity/Audit APIs Undocumented

**Behavior:**
```javascript
GET /tc/api/2.0/projects/{projectId}/activity
// May or may not exist
```

**Status:** Not in public documentation

**Evidence:**
- Trimble Connect UI shows activity feeds
- API endpoints likely exist but undocumented
- May require enterprise tier or special access

**Options:**
1. Inspect network traffic in Trimble Connect UI
2. Contact connect-support@trimble.com
3. Try common patterns (/activity, /audit, /history)
4. Accept limitation and work around it

---

## Performance Issues

### 15. Large File Lists Cause Slowdown

**Behavior:**
```javascript
// Project with 10,000+ files
const files = await fetchFiles(projectId);
// Request takes 30+ seconds
// UI freezes
```

**Reason:** API returns all files in single response

**Mitigation:**
1. Use pagination if available (check API docs)
2. Implement virtual scrolling in UI
3. Load files in background
4. Show loading indicator
5. Cache results

---

### 16. Multiple Concurrent API Calls May Hit Rate Limit

**Behavior:**
```javascript
// Fetching data for 50 projects simultaneously
await Promise.all(projects.map(p => fetchFiles(p.id)));
// Some requests fail with 429 Too Many Requests
```

**Reason:** Trimble Connect rate limiting (exact limits undocumented)

**Fix:** Implement request throttling

```javascript
async function throttleRequests(requests, maxConcurrent = 5) {
    const results = [];
    for (let i = 0; i < requests.length; i += maxConcurrent) {
        const batch = requests.slice(i, i + maxConcurrent);
        const batchResults = await Promise.all(batch.map(fn => fn()));
        results.push(...batchResults);
    }
    return results;
}

// Usage
const requests = projects.map(p => () => fetchFiles(p.id));
const results = await throttleRequests(requests, 5);
```

---

## Security Gotchas

### 17. Access Token Visible in Network Tab

**Behavior:**
```javascript
fetch(url, {
    headers: { 'Authorization': `Bearer ${token}` }
});
// Token visible in browser DevTools Network tab
```

**Implication:** Anyone with access to DevTools can see token

**Mitigation:**
- This is normal behavior for web apps
- Token is short-lived (expires)
- Don't log tokens to console
- Use HTTPS only
- Implement backend for sensitive operations

---

### 18. XSS Risk with User Input

**Behavior:**
```javascript
const project = await API.project.getProject();
element.innerHTML = project.name; // DANGER if name contains <script>
```

**Reason:** API responses may contain user-generated content

**Fix:** Always sanitize or use textContent

```javascript
// ❌ Vulnerable
element.innerHTML = project.name;

// ✅ Safe
element.textContent = project.name;

// ✅ Safe with sanitization
element.innerHTML = sanitizeHtml(project.name);
```

---

## Deployment Gotchas

### 19. Branch Mismatch (master vs main)

**Behavior:**
```
GitHub Actions workflow on 'main' branch
Local branch: 'master'
Git push: No deployment triggered
```

**Reason:** GitHub Actions listens to specific branch

**Fix:**
```bash
# Rename branch
git branch -m master main

# Push to new branch
git push -u origin main

# Update GitHub default branch in settings
```

---

### 20. Public Repository Required for GitHub Pages

**Behavior:**
```
Repository: Private
GitHub Pages: Enabled
Extension manifest URL: 404 Not Found
```

**Reason:** GitHub Pages free tier requires public repository

**Fix:** Make repository public or upgrade to GitHub Pro

---

## Testing Gotchas

### 21. Cannot Test Workspace API Locally

**Behavior:**
```
Open index.html in browser locally
TrimbleConnectWorkspace.connect() fails
```

**Reason:** Workspace API requires iframe context from Trimble Connect

**Workaround:**
```javascript
if (window.parent === window) {
    console.log('Running standalone - mocking API');
    // Implement mock API for local testing
    window.TrimbleConnectWorkspace = {
        connect: async () => ({
            project: {
                getProject: async () => ({
                    id: 'test-id',
                    name: 'Test Project',
                    location: 'us'
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

## Debugging Checklist

When plugin doesn't work:

1. **Check Workspace API loaded:**
   ```javascript
   console.log('API:', typeof TrimbleConnectWorkspace);
   ```

2. **Check connection:**
   ```javascript
   console.log('Connected:', !!globalAPI);
   ```

3. **Check token:**
   ```javascript
   console.log('Token:', accessToken ? 'present' : 'missing');
   ```

4. **Check region:**
   ```javascript
   const project = await API.project.getProject();
   console.log('Region:', project.location);
   ```

5. **Check API endpoint:**
   ```javascript
   console.log('API host:', getApiHost(project.location));
   ```

6. **Check browser console:**
   - Look for errors (red text)
   - Look for CORS errors
   - Look for 404/401/403 errors

7. **Check Network tab:**
   - Are requests being made?
   - What status codes?
   - What response body?

8. **Check GitHub Pages:**
   - Actions tab: Deployment successful?
   - Visit manifest URL directly in browser
   - Hard refresh (Ctrl+Shift+R)

---

## Summary: Top 5 Most Common Issues

1. ✅ Use `TrimbleConnectWorkspace` not `WorkspaceAPI`
2. ✅ Use `getPermission("accesstoken")` not `requestAccessToken()`
3. ✅ Use backend proxy for REST API (CORS blocks direct calls)
4. ✅ Map project region to correct API host (app21 for Europe)
5. ✅ Hard refresh browser to see changes (Ctrl+Shift+R)

**Follow these and avoid 90% of issues.**

---

## Next Steps

**For implementation:** Read [04_BASIC_PLUGIN.md](04_BASIC_PLUGIN.md)
**For code examples:** Read [08_CODE_TEMPLATES.md](08_CODE_TEMPLATES.md)
