# Deployment Guide for Frontend-Only Open WebUI

This guide will help you build and deploy the frontend-only version of Open WebUI to various hosting platforms.

## Building the Project

### Prerequisites

- Node.js 18+ and npm
- Git
- Basic command line knowledge

### Build Steps

1. Clone the repository
   ```bash
   git clone https://github.com/your-fork/open-webui.git
   cd open-webui
   ```

2. Install dependencies
   ```bash
   npm install
   ```

3. Build the frontend-only version
   ```bash
   # Using the build script
   ./scripts/build-static.sh
   
   # Or manually
   export FRONTEND_ONLY=true
   export VITE_BUILD_HASH=$(git rev-parse --short HEAD)
   npm run build:static
   ```

4. The built application will be in the `dist` directory

## Local Testing

To test the built application locally:

```bash
# Using npm
npx http-server dist

# Or using Python
cd dist
python -m http.server 8080
```

Then visit `http://localhost:8080` in your browser.

## Deployment Options

### GitHub Pages

1. Create a new repository on GitHub or use an existing one

2. Initialize git in your project (if not already)
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   ```

3. Add GitHub repository as remote
   ```bash
   git remote add origin https://github.com/username/repo-name.git
   ```

4. Set up GitHub Pages deployment
   ```bash
   # Create a dedicated branch for GitHub Pages
   git checkout -b gh-pages
   
   # Copy built files to root
   cp -r dist/* .
   
   # Add files to git
   git add .
   git commit -m "Add built files for GitHub Pages"
   
   # Push to GitHub
   git push -u origin gh-pages
   ```

5. In GitHub repository settings, enable GitHub Pages and select the `gh-pages` branch as the source

### Netlify

1. Create an account on Netlify if you don't have one

2. Install Netlify CLI
   ```bash
   npm install -g netlify-cli
   ```

3. Deploy to Netlify
   ```bash
   # Log in to Netlify
   netlify login
   
   # Deploy
   netlify deploy --prod --dir=dist
   ```

4. Alternatively, connect your GitHub repository to Netlify and configure it to:
   - Build command: `npm run build:static`
   - Publish directory: `dist`

### Vercel

1. Create an account on Vercel if you don't have one

2. Install Vercel CLI
   ```bash
   npm install -g vercel
   ```

3. Deploy to Vercel
   ```bash
   # Log in to Vercel
   vercel login
   
   # Deploy
   vercel --prod dist
   ```

4. Alternatively, connect your GitHub repository to Vercel and configure it to:
   - Build command: `npm run build:static`
   - Output directory: `dist`

### Firebase Hosting

1. Create a Firebase project if you don't have one

2. Install Firebase CLI
   ```bash
   npm install -g firebase-tools
   ```

3. Initialize Firebase in your project
   ```bash
   firebase login
   firebase init hosting
   ```

4. Configure Firebase to use the `dist` directory

5. Deploy to Firebase
   ```bash
   firebase deploy --only hosting
   ```

## Configuration for Single-Page Application Routing

Since Open WebUI is a single-page application, you'll need to configure your hosting provider to redirect all requests to `index.html`.

### For GitHub Pages, Netlify, or Vercel

Create a `_redirects` file in the `public` directory (before building):

```
/* /index.html 200
```

For GitHub Pages, you'll also need a `404.html` that redirects to `index.html`. This is included in the build script.

### For Firebase Hosting

In your `firebase.json`:

```json
{
  "hosting": {
    "public": "dist",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ]
  }
}
```

## CORS Configuration for API Calls

Since the frontend-only version makes direct API calls to LLM providers, you may encounter CORS issues. Here are some options:

1. **Configure a CORS proxy in your deployment**
   
   Add environment variables to point to a CORS proxy:
   ```
   VITE_CORS_PROXY=https://your-cors-proxy.com/
   ```

2. **Use a service like cors-anywhere**
   
   You can deploy your own instance of cors-anywhere or use an existing service.

3. **Note about API key security**
   
   When using a CORS proxy, be careful not to expose your API keys. Ensure your proxy doesn't log or store the requests.

## Post-Deployment Steps

After deployment, you should:

1. Test the deployment thoroughly
2. Check API connections work properly
3. Verify local storage is functioning
4. Test on different browsers and devices

## Updating Your Deployment

To update your deployment:

1. Make your changes to the code
2. Rebuild the project
   ```bash
   ./scripts/build-static.sh
   ```
3. Redeploy using the same method as your initial deployment

## Troubleshooting

### Common Issues

1. **API calls fail with CORS errors**
   - Solution: Use a CORS proxy or configure your API provider to allow your domain

2. **API calls fail with 401 Unauthorized**
   - Solution: Check that your API keys are correctly entered and stored

3. **Chat history disappears**
   - Solution: Check that you're not in incognito mode, which limits IndexedDB storage

4. **Application stuck on loading screen**
   - Solution: Check browser console for errors; clear browser cache and reload

### Browser Support

The frontend-only version has been tested on:
- Chrome 90+
- Firefox 90+
- Safari 14+
- Edge 90+

For older browsers, you may encounter issues with IndexedDB or other modern browser features.