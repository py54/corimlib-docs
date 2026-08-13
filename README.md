# CorimLib Documentation

Developer documentation for **CorimLib**, a cross-loader GUI/HUD/feature framework for Fabric, NeoForge, and Forge Minecraft mods. Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and deployed to GitHub Pages via GitHub Actions.

**This repository contains only documentation — no CorimLib source code.** The main CorimLib repository is private; this site documents its public API (verified directly against the real source) without reproducing it.

## Local development

```bash
pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>.

## Deployment

Pushing to `main` triggers `.github/workflows/deploy.yml`, which builds the site with MkDocs and deploys it to GitHub Pages automatically.
