# dbrownems_blog

Personal blog of David Browne, built with [Jekyll](https://jekyllrb.com/) and the
[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme, hosted on GitHub Pages.

Live site: <https://dbrownems.github.io>

## Writing a new post

1. Create a file under `_posts/` named `YYYY-MM-DD-title.md`.
2. Add front-matter:

   ```yaml
   ---
   title: My post title
   date: 2026-05-09 16:00:00 -0500
   categories: [Fabric, Notes]
   tags: [delta-lake, spark]
   ---
   ```

3. Write the content in Markdown.
4. `git add`, `git commit`, `git push` — the GitHub Actions workflow builds and deploys.

## Running locally (optional)

Requires Ruby 3.1+ and Bundler:

```bash
bundle install
bundle exec jekyll serve --livereload
```

Open <http://127.0.0.1:4000>.

## Deployment

`.github/workflows/pages-deploy.yml` builds the site on every push to `main` and
deploys to GitHub Pages. Make sure **Settings → Pages → Build and deployment →
Source** is set to **GitHub Actions**.
