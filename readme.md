# Local development

## Linux / WSL

Run:

```bash
docker run -it --rm -v "$PWD":/usr/src/app -p "4000:4000" starefossen/github-pages
```

Open a local browser and point at: `http://localhost:4000/`