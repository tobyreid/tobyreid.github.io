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
