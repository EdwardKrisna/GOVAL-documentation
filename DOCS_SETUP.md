# Documentation Setup Guide

This project uses **MkDocs Material** for beautiful documentation with sidebar navigation.

## 📋 Quick Start

### 1. Install Dependencies

```bash
pip install -r requirements-docs.txt
```

### 2. Preview Documentation Locally

Run the local development server:

```bash
mkdocs serve
```

Then open your browser to: **http://127.0.0.1:8000**

The site will auto-reload when you make changes to the documentation files.

### 3. Build Static Site

To build the documentation site:

```bash
mkdocs build
```

This creates a `site/` folder with the static HTML files.

---

## 🚀 Deploy to GitHub Pages

### Option 1: Automated Deployment

Deploy directly from command line:

```bash
mkdocs gh-deploy
```

This will:
- Build the documentation
- Push to the `gh-pages` branch
- Your site will be live at: `https://yourusername.github.io/GOVAL/`

### Option 2: Manual GitHub Actions

Create `.github/workflows/docs.yml`:

```yaml
name: Deploy Documentation

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: 3.x
      - run: pip install -r requirements-docs.txt
      - run: mkdocs gh-deploy --force
```

---

## 📁 Documentation Structure

```
GOVAL/
├── mkdocs.yml              # MkDocs configuration (navigation, theme, etc.)
├── docs/                   # Documentation source files
│   ├── index.md            # Home page
│   ├── about.md            # About page
│   ├── coming-soon.md      # Placeholder for future projects
│   └── jabodetabek/        # Jabodetabek project docs
│       ├── index.md        # Project overview (from README.md)
│       ├── workflow.md     # Workflow guide
│       ├── features.md     # Features & methods
│       ├── data-requirements.md
│       └── quick-reference.md
└── site/                   # Generated site (ignored by git)
```

---

## ✏️ Editing Documentation

### Adding New Pages

1. Create a new `.md` file in `docs/` folder
2. Add it to navigation in `mkdocs.yml`:

```yaml
nav:
  - Home: index.md
  - Your New Page: your-page.md
```

### Adding New Project

1. Create folder: `docs/your-project/`
2. Add markdown files
3. Update `mkdocs.yml` navigation:

```yaml
nav:
  - Your Project:
    - Overview: your-project/index.md
    - Guide: your-project/guide.md
```

### Markdown Features

MkDocs Material supports:

- **Admonitions** (tip, warning, note boxes)
- **Code highlighting**
- **Tables**
- **Tabs**
- **Buttons**

Example:

```markdown
!!! tip "Pro Tip"
    Use the search bar to quickly find topics!

[Click Here](link.md){ .md-button .md-button--primary }
```

---

## 🎨 Customization

### Change Theme Colors

Edit `mkdocs.yml`:

```yaml
theme:
  palette:
    primary: indigo  # Change to: red, blue, green, etc.
    accent: indigo
```

### Update Repository Links

Edit `mkdocs.yml`:

```yaml
repo_url: https://github.com/yourusername/GOVAL
repo_name: yourusername/GOVAL
```

---

## 📚 Resources

- [MkDocs Material Documentation](https://squidfunk.github.io/mkdocs-material/)
- [MkDocs Documentation](https://www.mkdocs.org/)
- [Markdown Guide](https://www.markdownguide.org/)

---

## ❓ Troubleshooting

### Port already in use

```bash
mkdocs serve -a 127.0.0.1:8001
```

### Module not found

```bash
pip install --upgrade -r requirements-docs.txt
```

### GitHub Pages not updating

1. Check if `gh-pages` branch exists
2. Enable GitHub Pages in repo Settings → Pages
3. Select `gh-pages` branch as source

---

**Happy documenting!** 📖
