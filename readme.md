# tobyreid.github.io — Local Development

This repo is a Jekyll site. You can run it locally either with Docker (recommended, zero Ruby setup) or with a native Ruby/Jekyll environment.

## Prerequisites

- Docker Desktop (for the Docker option)
- Or: Ruby + Bundler (for native Jekyll)

## Option A: Run with Docker

Runs GitHub Pages-compatible Jekyll in a container and serves on port 4000.

Linux / WSL:

```bash
docker run -it --rm -v "$PWD:/usr/src/app" -p "4000:4000" starefossen/github-pages
```

Windows (PowerShell):

```powershell
docker run -it --rm -v "${PWD}:/usr/src/app" -p "4000:4000" starefossen/github-pages
```

Windows (CMD):

```cmd
docker run -it --rm -v "%cd%:/usr/src/app" -p "4000:4000" starefossen/github-pages
```

Then open: `http://localhost:4000/`

## Option B: Run with Ruby/Jekyll

```powershell
bundle install
bundle exec jekyll serve
```

Open: `http://localhost:4000/`

## Theming

- Light mode is the default.
- Dark mode automatically applies for users with system/browser dark preference via `prefers-color-scheme`.

## Common Tips

- If port 4000 is busy, use `-p "4001:4000"` and browse `http://localhost:4001/`.
- Clear Jekyll cache if layouts don’t update: delete `.jekyll-cache` then re-run.
- Sass is compiled via Jekyll; changes in `css/pixyll.scss` and `_sass/` rebuild on save.

## Deploy

This is a GitHub Pages site. Push to `master` triggers deployment.

## Custom Domain & Config

- Custom domain: the file `CNAME` contains `www.tobyreid.co.uk`. GitHub Pages uses this to serve the site at that domain. Ensure your DNS has a CNAME record pointing `www.tobyreid.co.uk` to `tobyreid.github.io`.
- Site URL: `_config.yml` sets `url: "https://tobyreid.github.io"` and `baseurl: ""`. For canonical links at the custom domain, you can change `url` to `https://www.tobyreid.co.uk`.
- Sitemap & pagination: `_config.yml` enables `jekyll-sitemap` and `jekyll-paginate`.
- Analytics: both Universal Analytics (`UA-7044546-3`) and GA4 (`G-8HCDXG3B96`) IDs are present. Avoid enabling GTM simultaneously to prevent double-counting.
- Comments: Disqus shortname is `tobyreid`; comments render when enabled in layouts. Facebook comments are disabled.
- Social: usernames for GitHub, Twitter, LinkedIn are configured in `_config.yml`.
- Internationalization: Open Graph locale is set to `en_GB`.

### DNS Quick Reference
- `www` CNAME → `tobyreid.github.io`
- Optionally add `@` A records to GitHub Pages IPs if you plan to support apex domain (without `www`). GitHub recommends configuring this via the Pages settings using `ALIAS/ANAME` or A records to their IPs.

### GitHub Pages Settings
- In the repository Settings → Pages: set source to `master` (or `main`) branch, root.
- Add the custom domain `www.tobyreid.co.uk` and enable HTTPS.

## Novel/Important Customizations
- Theme: Pixyll, with a custom dark theme enabled via `prefers-color-scheme` (`_sass/_dark.scss` and imported from `css/pixyll.scss`).
- Performance/Build: Sass compiled by Jekyll (`sass.compressed: true`).
- Content features: anchors, related posts, social icons, and animations are configurable via flags in `_config.yml`.
