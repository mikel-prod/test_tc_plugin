# Hello World - Trimble Connect Extension

A simple "Hello World" extension for Trimble Connect, demonstrating how to create and deploy a custom extension.

## Overview

This extension displays a beautiful "Hello World" message and serves as a starting point for building Trimble Connect extensions.

## Features

- Clean, modern UI with gradient design
- Responsive layout
- Detects Trimble Connect Workspace API
- Ready for GitHub Pages deployment

## Project Structure

```
test_tc_plugin/
├── index.html          # Main extension UI
├── manifest.json       # Extension manifest (configuration)
├── icon.svg           # Extension icon
├── .github/
│   └── workflows/
│       └── deploy.yml # GitHub Actions deployment workflow
└── README.md          # This file
```

## Deployment Instructions

### Step 1: Create GitHub Repository

1. Go to [GitHub](https://github.com) and create a new repository named `test_tc_plugin`
2. Make it **public** (required for GitHub Pages free hosting)
3. Don't initialize with README (we already have one)

### Step 2: Update Manifest File

Before pushing to GitHub, update [manifest.json](manifest.json) with your GitHub username:

```json
{
  "url": "https://YOUR_GITHUB_USERNAME.github.io/test_tc_plugin/index.html",
  "title": "Hello World Extension",
  "icon": "https://YOUR_GITHUB_USERNAME.github.io/test_tc_plugin/icon.svg",
  "description": "A simple Hello World extension for Trimble Connect",
  "infoUrl": "https://github.com/YOUR_GITHUB_USERNAME/test_tc_plugin",
  "enabled": true
}
```

Replace `YOUR_GITHUB_USERNAME` with your actual GitHub username.

### Step 3: Push to GitHub

Run these commands in your terminal:

```bash
# If you haven't already initialized git
git init

# Add all files
git add .

# Create initial commit
git commit -m "Initial commit: Hello World Trimble Connect extension"

# Add your remote repository (replace with your repo URL)
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/test_tc_plugin.git

# Push to GitHub
git branch -M main
git push -u origin main
```

### Step 4: Enable GitHub Pages

1. Go to your repository on GitHub
2. Click **Settings** > **Pages** (in the left sidebar)
3. Under **Source**, select:
   - Source: **GitHub Actions**
4. The GitHub Action will automatically deploy your extension

Wait a few minutes for the deployment to complete. You can check the progress in the **Actions** tab.

### Step 5: Verify Deployment

Once deployed, your extension will be available at:
```
https://YOUR_GITHUB_USERNAME.github.io/test_tc_plugin/index.html
```

Open this URL in a browser to verify it works.

Your manifest file will be at:
```
https://YOUR_GITHUB_USERNAME.github.io/test_tc_plugin/manifest.json
```

### Step 6: Install in Trimble Connect

1. Log in to [Trimble Connect](https://connect.trimble.com/)
2. Open a project
3. Go to **Project Settings** > **Extensions**
4. In the **Extension manifest URL** field, paste:
   ```
   https://YOUR_GITHUB_USERNAME.github.io/test_tc_plugin/manifest.json
   ```
5. Click **Add** or **Install**
6. The extension should appear in the left navigation panel

## Testing Locally

To test the extension locally before deploying:

1. Open [index.html](index.html) directly in your browser
2. You'll see the Hello World message (Workspace API features won't work outside Trimble Connect)

For local development with a server:

```bash
# Using Python 3
python -m http.server 8000

# Using Python 2
python -m SimpleHTTPServer 8000

# Using Node.js (if you have http-server installed)
npx http-server -p 8000
```

Then visit: `http://localhost:8000/index.html`

## Customization

### Changing Colors

Edit the CSS gradient in [index.html](index.html):

```css
background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
```

### Adding Functionality

To interact with Trimble Connect, use the Workspace API:

```javascript
// Example: Get project information
if (window.TC) {
    window.TC.getProject().then(project => {
        console.log('Project name:', project.name);
    });
}
```

Refer to the [Trimble Connect Workspace API documentation](https://components.connect.trimble.com/trimble-connect-workspace-api/index.html) for available methods.

## Important Notes

### CORS Configuration

GitHub Pages automatically serves files with appropriate CORS headers, which is required for Trimble Connect to load your manifest file.

### HTTPS Required

Trimble Connect requires extensions to be served over HTTPS. GitHub Pages provides this automatically.

### File Updates

After making changes to your extension:

1. Commit and push to GitHub:
   ```bash
   git add .
   git commit -m "Update extension"
   git push
   ```

2. Wait for GitHub Actions to redeploy (usually 1-2 minutes)
3. Clear cache in Trimble Connect or reload the extension

## Resources

- [Trimble Connect Developer Documentation](https://developer.trimble.com/docs/connect/)
- [Workspace API Documentation](https://components.connect.trimble.com/trimble-connect-workspace-api/index.html)
- [Trimble Connect Extensions Guide](https://docs.browser.connect.trimble.com/projects/project-settings/extensions)
- [GitHub Pages Documentation](https://docs.github.com/en/pages)

## Troubleshooting

### Extension not loading in Trimble Connect

- Verify the manifest URL is correct and accessible
- Check that CORS is enabled (GitHub Pages does this automatically)
- Ensure your repository is public
- Check the browser console for error messages

### GitHub Pages not deploying

- Go to the **Actions** tab in your repository
- Check for any failed workflow runs
- Ensure GitHub Pages is enabled in repository settings
- Make sure the repository is public

### Extension shows "Standalone mode"

This is normal when opening the HTML file directly. The Workspace API is only available when running inside Trimble Connect.

## Next Steps

- Add real functionality using the Trimble Connect Workspace API
- Style the extension to match your needs
- Add user interactions (buttons, forms, etc.)
- Access project data, models, and files
- Implement advanced features

## License

MIT License - Feel free to use and modify for your projects.
