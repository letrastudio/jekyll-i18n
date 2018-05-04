## Plug-ins

I've tested the GitHub Pages plug-in trifecta with this site:

- `jekyll-sitemap` includes every page without a problem.
- `jekyll-feed` will only generate a feed for the default `posts` collection. If you want localized feeds, you can use liquid layouts like [jekyll-rss-feeds](https://github.com/snaptortoise/jekyll-rss-feeds) and [jekyll-json-feeds](https://github.com/snaptortoise/jekyll-json-feeds).
- `jekyll-seo-tag` works fine. `lang` must be set for every collection (using front matter defaults), or else everything will have `og:locale` set to `en-US` (or whatever `site.lang` is set to).

