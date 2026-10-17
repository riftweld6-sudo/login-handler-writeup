# The Login Handler That Believed Every Token

A sanitized vulnerability write-up published as a static GitHub Pages site.

- Repository: `riftweld6-sudo/login-handler-writeup`
- Published site: `https://riftweld6-sudo.github.io/login-handler-writeup/`
- Theme: `jekyll-theme-midnight`
- Build: GitHub Pages, branch `main`, root directory

## Local preview

No local build is required. If you want to preview the site with Jekyll:

```bash
bundle exec jekyll serve
```

## Update and publish

```bash
git add -A
git commit -m "Update write-up"
git push
```

GitHub Pages rebuilds automatically after each push to `main`.

## Configuration

`_config.yml` already points at the organization Pages site:

```yaml
url: "https://riftweld6-sudo.github.io"
baseurl: "/login-handler-writeup"
```

If the repository is renamed, update `baseurl` to match the new repository name.
