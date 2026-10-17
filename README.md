# One assertion to a session

A sanitized, lab-confined write-up of an identity-assertion validation bypass.

- **Live site:** https://riftweld6-sudo.github.io/login-handler-writeup/
- **Repository:** https://github.com/riftweld6-sudo/login-handler-writeup
- **Published article:** [`docs/index.html`](docs/index.html)
- **Markdown source:** [`WRITEUP.md`](WRITEUP.md)

## Structure

```text
.
├── README.md
├── WRITEUP.md
└── docs/
    ├── .nojekyll
    └── index.html
```

`docs/index.html` is a self-contained static page. It uses the same visual
language as the other RiftWeld publications: Sora and IBM Plex Mono, a warm
orange accent, a sticky table of contents, and a dark code surface.

## Publishing

GitHub Pages is configured to serve the `docs/` directory from `main`.

To update the published page:

```bash
# edit docs/index.html and WRITEUP.md
git add -A
git commit -m "Update write-up"
git push
```

Pages rebuilds automatically. The source is static HTML; no Jekyll build step
is required.

## Local preview

```bash
python3 -m http.server -d docs 8000
```

Then open `http://localhost:8000/`.

## Sanitization

All company names, domains, IP addresses, endpoints, user identifiers, and
customer data have been removed or replaced with synthetic values such as
`api.example.test` and `victim@example.test`. The proof of concept is
conceptual and omits actionable exploit material.
