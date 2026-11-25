# Senzo Sigcu: Security Writeups

Source for my cybersecurity lab write-ups site: Jekyll + the minima theme, hosted on GitHub Pages.

## Preview locally (optional)
```
bundle install
bundle exec jekyll serve
```
Then open <http://localhost:4000>.

## Publish
1. Push this repo to GitHub.
2. Settings → Pages → Build and deployment → Deploy from a branch → `main` / `root`.
The site builds automatically. If this is a **project** repo (`username.github.io/repo`),
set `baseurl: "/repo"` in `_config.yml`; a **user/org** repo (`username.github.io`) needs no change.

## Add a new write-up
1. Copy `writeups/metasploitable-2/` to `writeups/<your-target>/`.
2. Edit the front matter (`title`, `platform`, `skills`, `date`) and fill in each section.
3. Put screenshots in that write-up's `images/` folder.
It appears on the Write-ups page automatically. The listing picks up any page with
`writeup: true` in its front matter, newest first.
