# Trimble Connect Plugin Development Documentation Index

**Purpose:** This documentation enables AI agents and developers to create Trimble Connect plugins from scratch.

**Read Order:** Follow the numbered sequence for systematic understanding.

---

## Documentation Structure

### Core Concepts
1. [01_OVERVIEW.md](01_OVERVIEW.md) - Platform overview and plugin architecture
2. [02_WORKSPACE_API.md](02_WORKSPACE_API.md) - Workspace API capabilities and limitations
3. [03_REST_API.md](03_REST_API.md) - REST API endpoints and authentication

### Implementation
4. [04_BASIC_PLUGIN.md](04_BASIC_PLUGIN.md) - Step-by-step basic plugin creation
5. [05_ADVANCED_PLUGIN.md](05_ADVANCED_PLUGIN.md) - Backend integration patterns
6. [06_DEPLOYMENT.md](06_DEPLOYMENT.md) - Deployment to GitHub Pages and other platforms

### Reference
7. [07_QUIRKS_AND_GOTCHAS.md](07_QUIRKS_AND_GOTCHAS.md) - Common issues and solutions
8. [08_CODE_TEMPLATES.md](08_CODE_TEMPLATES.md) - Reusable code templates
9. [09_API_ENDPOINTS.md](09_API_ENDPOINTS.md) - API endpoint reference

---

## Quick Start Paths

### Path 1: Simple Display Plugin (No Backend)
Read: 01 → 02 → 04 → 06 → 07

**Capabilities:**
- Display project information
- Show user settings
- Embed Trimble's viewers
- No file access, no data processing

**Example:** Project info display, user preference viewer

---

### Path 2: Advanced Plugin with Backend
Read: 01 → 02 → 03 → 05 → 06 → 07

**Capabilities:**
- Full REST API access
- File listing and processing
- Data analytics
- Custom storage
- Background processing

**Example:** File statistics, BIM validator, activity tracker

---

## Key Findings Summary

### ✅ What Works
- Static HTML/JS plugins hosted on GitHub Pages (free)
- Workspace API for basic project/user info
- Menu integration in Trimble Connect sidebar
- Access token retrieval via permission request
- Embedded viewers (3D, file explorer, project list)

### ❌ What Doesn't Work
- Direct REST API calls from browser (CORS blocked)
- File listing without backend proxy
- Activity/audit log access (undocumented)
- Service account authentication (requires user OAuth)

### ⚠️ Critical Quirks
- Global variable is `TrimbleConnectWorkspace` NOT `WorkspaceAPI`
- Regional endpoints vary (US, Europe, APAC have different hosts)
- Access token method is `getPermission("accesstoken")` NOT `requestAccessToken()`
- Manifest must be publicly accessible with CORS enabled
- Extensions run in iframes with limited browser APIs

---

## Common Use Cases

| Use Case | Backend Required | Complexity | Read Path |
|----------|------------------|------------|-----------|
| Display project metadata | No | Low | Path 1 |
| Embed 3D viewer | No | Low | Path 1 |
| List project files | Yes | Medium | Path 2 |
| File statistics/analytics | Yes | Medium | Path 2 |
| BIM model validation | Yes | High | Path 2 |
| Activity tracking | Yes* | High | Path 2 |
| Multi-project reports | Yes | High | Path 2 |

*Activity tracking API is undocumented; may require reverse engineering or Trimble support contact.

---

## External Resources

- [Trimble Connect Developer Portal](https://developer.trimble.com/docs/connect/)
- [Workspace API Documentation](https://components.connect.trimble.com/trimble-connect-workspace-api/index.html)
- [Example Repository](https://github.com/mikel-prod/test_tc_plugin)

---

## Document Maintenance

**Last Updated:** 2025-12-17
**API Version:** Trimble Connect API 2.0
**Workspace API Version:** Latest (loaded from CDN)

**Note to AI Agents:** If documentation references become outdated, prioritize:
1. Official Trimble Developer Portal
2. Workspace API CDN documentation
3. This documentation (may be outdated)
