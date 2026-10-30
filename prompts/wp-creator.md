# WordPress Context (`{{SITE}}`)

## Paths
- **Preview**: `http://localhost:{{PORT}}` (`sites/{{SITE}}/wp-content` mounted; media proxied from `https://{{SITE}}`)
- **Posts / Essays**: `sites/{{SITE}}/content/posts/*.md`
- **Pages**: `sites/{{SITE}}/content/pages/*.md`
- **Themes**: `sites/{{SITE}}/wp-content/themes/`

## Content Rules (`content/posts/*.md`)
- Required YAML frontmatter:
  ```yaml
  ---
  title: TITLE
  slug: slug-name
  status: publish # or draft
  date: YYYY-MM-DDTHH:MM:SS
  categories:
      - TEXTE
  tags:
      - Tag
  ---
  ```
- Include `Lesezeit: X Minuten` at top of essays (~200 words/min).
- Proofreading: fix spelling/grammar only unless rewrite is requested.

## Theme / CSS Rules
- Edit active theme inside `sites/{{SITE}}/wp-content/themes/`.
- Never read or modify `wp-config.php` or `.env`.

## Deployment Commands (Requires User Approval)
- Push posts: `./wpctl post push {{SITE}}`
- Push pages: `./wpctl page push {{SITE}}`
- Deploy theme/files: `./wpctl deploy {{SITE}}`
