# 🐍 Python Interview Handbook

A single, high-level study website for senior Python / backend interviews:

| Section | What's inside |
|---|---|
| **Python** | Fundamentals, data types, functions, OOP, decorators, generators, context managers, descriptors, metaclasses, internals (GIL, memory, GC), concurrency & asyncio, testing & typing |
| **APIs** | HTTP, REST design, status codes, idempotency, pagination, versioning, FastAPI, DRF, auth (JWT/OAuth2), security, rate limiting |
| **Databases** | SQL from basics to window functions & CTEs, normalization, indexing, transactions & isolation, NoSQL, caching, ORM pitfalls |
| **Design** | SOLID, GoF design patterns in Python, data modeling, system design |
| **DSA** | Big-O, every core data structure, sorting/searching/graph/DP algorithms, problem-solving patterns |
| **Cheatsheet** | Last-day revision page |

## Run locally

```bash
pip install -r requirements.txt
mkdocs serve        # http://127.0.0.1:8000
```

## Publish on GitHub Pages

1. Push this repo to GitHub (branch `main`).
2. In the repo go to **Settings → Pages → Build and deployment → Source** and choose **GitHub Actions**.
3. Every push to `main` rebuilds and deploys the site via `.github/workflows/deploy.yml`.

Your site will be at `https://<your-username>.github.io/<repo-name>/`.

## License

MIT
