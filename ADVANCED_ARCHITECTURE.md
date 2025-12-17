# Advanced Trimble Connect Extension Architecture

This guide explains how to build more advanced Trimble Connect extensions with backend server integration.

## Current Simple Extension Architecture

**What you have now:**
```
Browser Extension (GitHub Pages)
    ↓
Trimble Connect Workspace API
    ↓
Limited data access (project info, user settings)
```

**Limitations:**
- CORS restrictions prevent direct API calls from browser
- Can only access data exposed through Workspace API
- No file listing, limited model interaction
- No persistent storage or background processing

---

## Advanced Extension Architecture

### Architecture Pattern: Frontend + Backend

```
┌─────────────────────────────────────────────────────────────┐
│  Trimble Connect Web UI                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Your Extension (GitHub Pages)                         │ │
│  │  - User Interface                                      │ │
│  │  - Workspace API integration                           │ │
│  │  - Gets access token from Workspace API                │ │
│  │  └──────────────────────────────────────────────────┐  │ │
│  └────────────────────────────────────────────────────┼──┘ │ │
└─────────────────────────────────────────────────────┼─────┘ │
                                                      │         │
                                   Access Token       │         │
                                                      ▼         │
┌────────────────────────────────────────────────────────────┐│
│  Your Backend Server (Vercel/AWS/Azure/etc.)              ││
│  ┌──────────────────────────────────────────────────────┐ ││
│  │  API Proxy Layer                                     │ ││
│  │  - Receives token from frontend                      │ ││
│  │  - Makes authenticated API calls                     │ ││
│  │  - Handles CORS                                      │ ││
│  │                                                       │ ││
│  │  Business Logic Layer                                │ ││
│  │  - Data processing                                   │ ││
│  │  - File analysis (statistics, validation, etc.)     │ ││
│  │  - Complex computations                              │ ││
│  │                                                       │ ││
│  │  Database/Cache (optional)                           │ ││
│  │  - Store computed results                            │ ││
│  │  - Cache API responses                               │ ││
│  │  - User preferences                                  │ ││
│  └──────────────────────────────────────────────────────┘ ││
└─────────────────────────────────┼──────────────────────────┘│
                                  │                            │
                                  ▼                            │
┌─────────────────────────────────────────────────────────────┤
│  Trimble Connect REST APIs                                   │
│  - Core API (files, folders, projects)                      │
│  - Model API (BIM queries)                                  │
│  - Organizer API (hierarchies)                              │
│  - Topics API (BCF issues)                                  │
│  - Property Set API                                         │
└──────────────────────────────────────────────────────────────┘
```

---

## Key Differences: Simple vs Advanced Extensions

### 1. **Data Access**

**Simple (Current):**
- ✅ Project metadata (name, ID, region)
- ✅ User settings
- ❌ File listings
- ❌ File contents
- ❌ Model queries
- ❌ Topics/Issues
- ❌ Custom data storage

**Advanced (With Backend):**
- ✅ Full project data access
- ✅ List and download files
- ✅ Upload files
- ✅ Query BIM models
- ✅ Manage BCF issues/topics
- ✅ Read/write property sets
- ✅ Custom database for analytics
- ✅ Background processing

### 2. **Capabilities**

**Simple:**
- Display project information
- Embed Trimble's viewers (3D, file explorer)
- Respond to user interactions
- Basic UI operations

**Advanced:**
- Process and analyze project files
- Generate reports and statistics
- Validate data against rules
- Sync with external systems
- Automated workflows
- Notifications and alerts
- Multi-project analytics

### 3. **Architecture Components**

**Simple:**
- Single HTML file + JavaScript
- Hosted on GitHub Pages (static)
- No server-side logic
- No database

**Advanced:**
- Frontend: HTML/React/Vue
- Backend: Node.js/Python/.NET API
- Database: PostgreSQL/MongoDB/Redis
- Hosting: Vercel/AWS/Azure/Google Cloud
- Optional: Message queue, cron jobs, webhooks

---

## Backend Server Implementation Guide

### Option 1: Serverless Functions (Recommended for Starting)

**Platforms:** Vercel, Netlify, Cloudflare Workers, AWS Lambda

**Example: Vercel Serverless Function**

```javascript
// api/proxy.js (Vercel serverless function)
export default async function handler(req, res) {
  // Enable CORS for your extension
  res.setHeader('Access-Control-Allow-Origin', 'https://mikel-prod.github.io');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type');

  if (req.method === 'OPTIONS') {
    return res.status(200).end();
  }

  // Get access token from request
  const accessToken = req.headers.authorization?.replace('Bearer ', '');

  if (!accessToken) {
    return res.status(401).json({ error: 'No access token' });
  }

  // Extract parameters
  const { projectId, region } = req.query;

  // Determine API host based on region
  const apiHost = region === 'europe'
    ? 'app21.connect.trimble.com'
    : region === 'apac'
    ? 'app31.connect.trimble.com'
    : 'app.connect.trimble.com';

  try {
    // Make request to Trimble Connect API
    const apiUrl = `https://${apiHost}/tc/api/2.0/projects/${projectId}/files`;
    const response = await fetch(apiUrl, {
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Accept': 'application/json'
      }
    });

    if (!response.ok) {
      throw new Error(`API returned ${response.status}`);
    }

    const data = await response.json();

    // Process data (example: count files by extension)
    const stats = processFileStatistics(data);

    return res.status(200).json(stats);

  } catch (error) {
    console.error('Error:', error);
    return res.status(500).json({ error: error.message });
  }
}

function processFileStatistics(files) {
  const extensionCounts = {};
  let totalFiles = 0;

  function processFiles(fileList) {
    if (!Array.isArray(fileList)) return;

    fileList.forEach(file => {
      if (file.type === 'FILE') {
        totalFiles++;
        const ext = file.name.split('.').pop().toLowerCase() || 'no extension';
        extensionCounts[ext] = (extensionCounts[ext] || 0) + 1;
      }
    });
  }

  processFiles(files);

  return {
    extensionCounts,
    totalFiles,
    timestamp: new Date().toISOString()
  };
}
```

**Frontend code to call the serverless function:**

```javascript
async function loadFileStatistics() {
  try {
    const project = await globalAPI.project.getProject();

    // Call your backend serverless function
    const response = await fetch(
      `https://your-project.vercel.app/api/proxy?projectId=${project.id}&region=${project.location}`,
      {
        headers: {
          'Authorization': `Bearer ${accessToken}`
        }
      }
    );

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const stats = await response.json();
    displayStatistics(stats.extensionCounts, stats.totalFiles);

  } catch (error) {
    console.error('Error:', error);
    showError('Failed to load statistics: ' + error.message);
  }
}
```

### Option 2: Full Backend Server

**Platforms:** AWS EC2, Azure App Service, Google Cloud Run, DigitalOcean

**Tech Stacks:**
- Node.js + Express
- Python + FastAPI/Flask
- .NET Core + ASP.NET
- Go + Gin/Echo

**Example: Express.js Server**

```javascript
// server.js
const express = require('express');
const cors = require('cors');
const fetch = require('node-fetch');

const app = express();

// CORS configuration
app.use(cors({
  origin: 'https://mikel-prod.github.io',
  credentials: true
}));

app.use(express.json());

// Proxy endpoint for file statistics
app.get('/api/projects/:projectId/file-stats', async (req, res) => {
  try {
    const { projectId } = req.params;
    const { region } = req.query;
    const accessToken = req.headers.authorization?.replace('Bearer ', '');

    if (!accessToken) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    // Determine API host
    const apiHost = getApiHost(region);

    // Fetch files from Trimble Connect
    const apiUrl = `https://${apiHost}/tc/api/2.0/projects/${projectId}/files`;
    const response = await fetch(apiUrl, {
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Accept': 'application/json'
      }
    });

    if (!response.ok) {
      throw new Error(`Trimble API error: ${response.status}`);
    }

    const files = await response.json();

    // Process and analyze files
    const stats = analyzeFiles(files);

    // Optional: Store in database
    await saveStatistics(projectId, stats);

    res.json(stats);

  } catch (error) {
    console.error('Error:', error);
    res.status(500).json({ error: error.message });
  }
});

// Additional endpoints for advanced features
app.post('/api/projects/:projectId/analyze', async (req, res) => {
  // Complex file analysis
  // Model validation
  // Custom business logic
});

app.get('/api/projects/:projectId/history', async (req, res) => {
  // Historical data from your database
});

function getApiHost(region) {
  const hosts = {
    'europe': 'app21.connect.trimble.com',
    'apac': 'app31.connect.trimble.com',
    'us': 'app.connect.trimble.com'
  };
  return hosts[region] || hosts.us;
}

function analyzeFiles(files) {
  // Your analysis logic here
  return {
    extensionCounts: {},
    totalFiles: 0,
    totalSize: 0,
    // ... more stats
  };
}

async function saveStatistics(projectId, stats) {
  // Save to database for historical tracking
}

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

## Advanced Features You Can Build

### 1. **File Analytics Dashboard**
- Track file changes over time
- Show file size distribution
- Identify duplicate files
- Monitor upload/download activity
- Storage usage by team member

### 2. **BIM Model Validator**
- Query IFC/BIM models using Model API
- Validate against project standards
- Check for missing properties
- Generate compliance reports
- Automated quality checks

### 3. **Issue Management Integration**
- Sync BCF topics with external systems (Jira, Azure DevOps)
- Automated issue assignment based on rules
- Email notifications for new issues
- Custom workflows and approvals
- Analytics on issue resolution time

### 4. **Document Management**
- Automated file organization
- Version control tracking
- Document templates
- Batch file operations
- Full-text search across documents

### 5. **Multi-Project Analytics**
- Aggregate data across multiple projects
- Portfolio-level reporting
- Resource allocation tracking
- Cross-project comparisons
- Executive dashboards

### 6. **Automated Workflows**
- File processing pipelines
- Scheduled reports
- Backup automation
- Format conversions
- Data exports to other systems

---

## Deployment Options Comparison

| Platform | Type | Cost | Setup | Best For |
|----------|------|------|-------|----------|
| **Vercel** | Serverless | Free tier generous | ⭐⭐⭐ Easy | Quick prototypes, low traffic |
| **Netlify** | Serverless | Free tier good | ⭐⭐⭐ Easy | Static + functions |
| **Cloudflare Workers** | Edge compute | Free tier excellent | ⭐⭐ Medium | Global low-latency |
| **AWS Lambda** | Serverless | Pay per use | ⭐⭐ Medium | AWS ecosystem |
| **Google Cloud Run** | Containers | Pay per use | ⭐⭐ Medium | Containerized apps |
| **Heroku** | PaaS | Paid (free tier removed) | ⭐⭐⭐ Easy | Full apps |
| **DigitalOcean** | VPS | $5+/month | ⭐ Complex | Full control |
| **Azure App Service** | PaaS | Pay per hour | ⭐⭐ Medium | Microsoft stack |

---

## Security Best Practices

### 1. **Token Handling**
```javascript
// ❌ NEVER store tokens in frontend localStorage/sessionStorage
// ❌ NEVER log tokens to console in production
// ✅ Always pass tokens in Authorization header
// ✅ Let backend validate tokens
// ✅ Use short-lived tokens when possible

// Frontend: Pass token to backend
fetch('https://your-api.com/endpoint', {
  headers: {
    'Authorization': `Bearer ${accessToken}`
  }
});

// Backend: Validate and use token
const token = req.headers.authorization?.replace('Bearer ', '');
if (!token || !isValidToken(token)) {
  return res.status(401).json({ error: 'Unauthorized' });
}
```

### 2. **CORS Configuration**
```javascript
// ✅ Whitelist specific origins
app.use(cors({
  origin: ['https://mikel-prod.github.io', 'https://your-domain.com'],
  credentials: true
}));

// ❌ Don't use wildcard in production
app.use(cors({ origin: '*' })); // INSECURE
```

### 3. **Input Validation**
```javascript
// ✅ Validate all inputs
app.get('/api/projects/:projectId', (req, res) => {
  const { projectId } = req.params;

  // Validate project ID format
  if (!/^[a-zA-Z0-9-]+$/.test(projectId)) {
    return res.status(400).json({ error: 'Invalid project ID' });
  }

  // Continue with validated input
});
```

### 4. **Rate Limiting**
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

app.use('/api/', limiter);
```

---

## Next Steps

1. **Choose your backend platform** (Start with Vercel for simplicity)
2. **Implement basic proxy endpoint** (File statistics as proof of concept)
3. **Test with your extension**
4. **Add database for persistence** (if needed)
5. **Implement advanced features** based on your requirements
6. **Add monitoring and logging** (Sentry, LogRocket, etc.)

---

## Example: Complete File Statistics with Vercel

I can help you set this up right now if you want. The steps would be:

1. Create a new Vercel project
2. Add the serverless function (api/proxy.js)
3. Update your extension to call the backend
4. Deploy to Vercel (free tier is sufficient)

Would you like me to implement this?

---

## Resources

- [Trimble Connect Developer Portal](https://developer.trimble.com/docs/connect/)
- [Workspace API Documentation](https://components.connect.trimble.com/trimble-connect-workspace-api/index.html)
- [Vercel Serverless Functions](https://vercel.com/docs/functions)
- [Netlify Functions](https://www.netlify.com/products/functions/)
- [Cloudflare Workers](https://workers.cloudflare.com/)
