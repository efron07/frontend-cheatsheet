# Frontend Cheat Sheet — JavaScript to Next.js (PharmaCare Edition)

JavaScript → TypeScript → HTML/CSS & Tailwind → React → Next.js, plus SEO, performance,
testing, security and accessibility. Plain-language explanations with working code,
all built around one example project: the PharmaCare pharmacy system.

**Live site:** https://efron07.github.io/frontend-cheatsheet/

## Edit
Pages live in `docs/`, one file per section. Edit and push to `main`; GitHub Actions
republishes in about a minute. Or click the pencil icon on any page of the live site.

## Preview locally
```bash
python3 -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
mkdocs serve        # http://127.0.0.1:8000
```

MkDocs is pinned to 1.6.x on purpose: MkDocs 2.0 breaks Material-based sites.
