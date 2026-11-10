# Deploying to Azure Static Web Apps

The repo is wired to deploy automatically on push to `main` via GitHub Actions.
To set it up the first time:

## 1. Create the Static Web App in Azure

Either from the Azure Portal (`+ Create a resource` → Static Web App) or via the
Azure CLI:

```bash
az staticwebapp create \
  --name tornyos-marketing \
  --resource-group <your-rg> \
  --source https://github.com/tornyglory/tornycrm-astro \
  --branch main \
  --app-location "/" \
  --output-location "dist" \
  --login-with-github
```

Pick the **Free** plan to start — it covers 100 GB bandwidth, custom domains,
and SSL.

If you let Azure connect to GitHub for you, it will also commit a workflow file
automatically — **delete** the one it generates, since the one in
`.github/workflows/azure-static-web-apps.yml` already has what we need
(Node 22 + skip-Oryx build).

## 2. Add the deployment token as a GitHub secret

After the SWA is created, grab the deployment token:

```bash
az staticwebapp secrets list \
  --name tornyos-marketing \
  --resource-group <your-rg> \
  --query "properties.apiKey" -o tsv
```

Azure auto-adds this secret when the SWA is created via GitHub integration
(name format: `AZURE_STATIC_WEB_APPS_API_TOKEN_<SWA-NAME>_<SUFFIX>`). The
current workflow references
`AZURE_STATIC_WEB_APPS_API_TOKEN_AMBITIOUS_GLACIER_038F7EF10`.

If you create the SWA a different way (CLI, or without GitHub integration),
set the secret manually:

```bash
gh secret set AZURE_STATIC_WEB_APPS_API_TOKEN_AMBITIOUS_GLACIER_038F7EF10 \
  --repo tornyglory/tornycrm-astro
```

(Paste the token when prompted.)

## 3. Push

The workflow fires on every push to `main` and every pull request:

- **Push to main** → deploys to production
- **Pull request** → spins up a preview URL (shown as a PR comment)
- **PR closed** → tears down the preview

## What the workflow does

1. Checks out the repo
2. Sets up Node 22
3. `npm ci` + `npm run build` (outputs `dist/`)
4. Uploads `dist/` to Azure SWA with `skip_app_build: true` (avoids Azure's
   Oryx builder — we build ourselves so the Node version and deps are exact)

## Config

- `public/staticwebapp.config.json` — SWA runtime config (headers, cache,
  404 fallback, MIME types). Astro copies it into `dist/` at build time.
- `.nvmrc` pins Node 22 for anyone cloning locally.

## Custom domain

Once the SWA is live, add a custom domain in Azure Portal → Static Web App →
Custom domains. Azure provisions the SSL cert automatically within ~15 min.

Update `astro.config.mjs` with the final URL so canonical links / sitemaps use it:

```js
export default defineConfig({
  site: 'https://torny.co',
});
```
