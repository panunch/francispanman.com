# francispanman.com

Portfolio, freeCodeCamp project showcase, dashboards and (later) an agentic lab, deployed as the `fp-www` Cloudflare Worker with Static Assets. Full design: [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md).

## Local development

```bash
npm install
npm run dev        # http://localhost:8787
```

Secrets: copy `.dev.vars.example` to `.dev.vars` for local values; use `npx wrangler secret put NAME` for production. Never commit real secrets.

## Deploy

Pushing to `main` deploys via Workers Builds (set up in session 0.3). Manual deploy: `npm run deploy`.

## Layout

```
public/                  static assets (served free from the edge)
  index.html             home
  404.html               served for unknown paths (not_found_handling)
  _headers               security + cache headers
  assets/css/site.css    shared design tokens
  projects/<slug>/       one self-contained folder per fCC project (relative paths only)
  dashboards/<slug>/     GIS / ops / analytics dashboards
  data/                  projects.json, changelog.json, simplified GeoJSON
src/worker.js            /api/* router (Phase 1b)
migrations/              D1 schema (Phase 1b)
tools/                   offline Python preprocessing (Phase 2)
lab/, mcp/               separate Workers (Phase 3)
```

## Adding an fCC project

1. Copy the project folder to `public/projects/<slug>/`.
2. Add an entry to `public/data/projects.json`.
3. Commit on a branch → check the Preview URL → merge.
