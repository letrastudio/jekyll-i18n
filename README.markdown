# jekyll-vanilla-i18n

A sane, plugin-free solution for internationalized Jekyll sites.

Minimum required Jekyll version is 3.7.0, or 3.8.0 if using `collections_dir`.



## Configuration and content organization

Locales are configured in `_config.yml`:

```yaml
locales:
  default:
    lang: en-US
    name: English
  pt:
    lang: pt-PT
    name: Português
```

The `default` locale should be the first one. Typically, that’s the one that builds directly to the site root. Other locales should use their key label as a baseurl — in this example, the `pt` locale would build to `/pt`. To override this behavior, a `baseurl` attribute can be set for each locale, including the default one.

The attributes in this example are both optional, but useful. `lang` should be a valid [Unicode Language Identifier](https://unicode.org/cldr/utility/languageid.jsp) and is primarily used to set the HTML `lang` attribute. `name` is used for naming links in the [language switcher element](/_includes/lang-switcher.html).

For every content type in your site, you need to create multiple mirrored collections — one for each locale. Collections in the `default` locale are named normally:

```yaml
collections:
  pages:
    output: true
    permalink: /:path/
  posts:
    output: true
    permalink: /blog/:categories/:title/
  photos:
    output: true
    permalink: /photos/:path/
```

Collections for other locales should be suffixed with an underscore and that locale’s label. For example, `posts` matches `posts_pt`. Permalinks need to be set manually for each collection to match the locale label (or the locale’s `baseurl` attribute, if set).

```yaml
  pages_pt:
    output: true
    permalink: /pt/:path/
  posts_pt:
    output: true
    permalink: /pt/blog/:categories/:title/
  photos_pt:
    output: true
    permalink: /pt/fotografias/:path/
```

In addition, front matter defaults need to be configured so that `page.locale` is always set. It’s also useful to set `lang` attributes here so that the [jekyll-seo-tag plugin](https://github.com/jekyll/jekyll-seo-tag) can pick them up.

```yaml
defaults:
- scope:
    path: '*_pt/'
  values:
    locale: pt
    lang: pt-PT
- scope:
    path: ''
  values:
    locale: default
    lang: en
```

Because using a `pages` collection for pages is not standard, `index` documents don’t quite work as normal. For each locale’s `index.markdown` (or `index.html`), a `permalink` must be set manually. This can be done in the documents themselves, or through more front matter defaults:

```yaml
- scope:
    path: '_pages/index.*'
  values:
    permalink: "/"
- scope:
    path: '_pages_pt/index.*'
  values:
    permalink: "/pt/"
```

Your site’s content should look something like this:

```
├── _pages
│   ├── about.markdown
│   ├── blog.markdown
│   ├── index.markdown
│   └── photos.markdown
├── _pages_pt
│   ├── about.markdown
│   ├── blog.markdown
│   ├── index.markdown
│   └── photos.markdown
├── _photos
│   ├── photo-1.markdown
│   └── photo-2.markdown
├── _photos_pt
│   ├── photo-1.markdown
│   └── photo-2.markdown
├── _posts
│   └── 2018-01-01-hello.markdown
└── _posts_pt
    └── 2018-01-01-hello.markdown
```

If you want to localize your permalinks, feel free to use different filenames for each locale. We’ll add front matter to match them up later.

```
├── _pages
│   ├── about.markdown
│   ├── blog.markdown
│   ├── index.markdown
│   └── photos.markdown
├── _pages_pt
│   ├── blog.markdown
│   ├── fotografias.markdown
│   ├── index.markdown
┆   └── sobre.markdown
```

It’s not necessary for each locale to have a copy of every collection, or every file — content can be asymmetric.



## Content matching between locales

Content matching is automatic when file and folder names are exactly the same. In this example both pages represent the same content:

```
├── _pages
│   └── subdir
│       └── page.markdown
└── _pages_pt
    └── subdir
        └── page.markdown
```

To match them, the `i18n.html` include generates a variable called `content_id`, based on each document’s path inside the collection.

Both pages in this example would return the following `content_id`:

```
/subdir/page
```

When filenames don’t match, you can set `content_id` manually via YAML front matter. If your content looks like this:

```
├── _pages
│   └── subdir
│       └── page.markdown
└── _pages_pt
    └── subpasta
        └── pagina.markdown
```

By adding this front matter to `pagina.markdown`, they’ll match:

```yaml
---
content_id: /subdir/page
---
```

This is better than using matching filenames and setting URL localizations through `page.permalink` because it’s derived from your document’s path in your site source, rather than the compiled site. That means you can change your global permalink style without having to edit your documents.

The initial `/` character is optional. These would match:

```
/about
about
```



## Working with i18n in Liquid

The `i18n.html` include does most of the heavy lifting for making i18n manageable in Liquid. It defines the following variables:

| Variable | Description |
|:--|:--|
| `locale` | Contains the attributes of the current locale. The same as `site.locales[page.locale]`. |
| `locale_baseurl` | An empty string if the current locale is `default`, or a slash followed by the current locale’s label otherwise (e.g. `/pt`) . Setting a `baseurl` attribute for the locale in `_config.yml` overrides this value. |
| `locale_strings` | Contains any localized text strings you’ve defined. More about that later. |
| `content_id` | Returns either the manually set or automatically generated `content_id` string for the page. |
| `localized_pages` | An array of objects for all pages matched through `content_id`, including the current page. |
| `default_page` | The page object of a matched `default` locale version of the current page. Is `nil` if a match isn’t found. Useful for falling back to the default locale when dealing with unlocalized content. |

`i18n.html` must be included at the top of every layout requiring i18n features:

```liquid
{% include i18n.html %}
```

When it’s useful to get i18n variables for a page other than the current one, the `obj` parameter can be passed. It’s important to note that this overwrites the current page’s variables. To restore them, the include must be called again. For example:

```liquid
<h2>Collection Document IDs</h2>
{% for document in collection %}
{% include i18n.html obj=document %}
<p>{{ content_id }}</p>
{% endfor %}

<h2>Current Page ID</h2>
{% include i18n.html %}
<p>{{ content_id }}</p>
```



## Content fallbacks

You can use `default_page` to fallback to content in the default locale if it’s not localized:

```yaml
{{ page.image | default: default_page.image }}
```

You may even use `localized_pages` to set up a fallback cascade:

```yaml
{{ page.image | default: localized_pages.pt.image | default: default_page.image }}
```

Using these fallbacks means that each localized file only needs to contain localizable content. The `about.markdown` file in the `default` locale might look like this:

```markdown
---
title: About
email: hello@example.com
phone: +99 123 456 789
image: logo.png
---
We are a cool company.
```

But the localized `sobre.markdown` in the `pt` locale can include just the content that requires localization:

```markdown
---
title: Sobre
content_id: about
---
Somos uma empresa fixe.
```



## Localizing text strings

Text strings are defined in `_data` — one YAML file for each locale, named `strings_LOCALE.yml`. The `default` locale file is named `strings.yml`.

```
└── _data
    ├── strings_pt.yml
    └── strings.yml
```

Any keys you set in these files can be accessed directly through `locale_strings`:

```liquid
{{ page.locale }}: {{ locale_strings.hello }}
```

```
default: Hello
pt: Olá
```



## Localizing dates

A special include deals with dates: `localized-date.html`.

Locale-specific date formats can be set in `strings.yml` data files using the `date_formats` key:

```yaml
date_formats:
  full: "%A, %e %B %Y"
  long: "%B %e, %Y"
  short: "%m/%d/%Y"
```

Month and weekday translations can also be set:

```yaml
date_formats:
  full: "%A, %e de %B de %Y"
  long: "%e %B %Y"
  short: "%d-%m-%Y"
  months:
    January: Janeiro
    February: Fevereiro
    March: Março
    ...
  weekdays:
    Sunday: Domingo
    Monday: Segunda-feira
    Tuesday: Terça-feira
    ...
```

The include can then be used by passing `date` and `format` parameters. If no format is specified, `%Y-%m-%d` will be used.

```
{% include localized-date.html date=page.date format="long" %}
```

```
default: September 15, 2016
pt: 15 Setembro 2016
```



## Plugin compatibility

I've tested the Jekyll plug-in trinity with this site:

- `jekyll-sitemap` includes every page without a problem, but they aren't marked up using [rel="alternate" links](https://support.google.com/webmasters/answer/2620865?hl=en).
- `jekyll-feed` will only generate a feed for the default `posts` collection. If you want localized feeds, you can use liquid layouts like [jekyll-rss-feeds](https://github.com/snaptortoise/jekyll-rss-feeds) and [jekyll-json-feeds](https://github.com/snaptortoise/jekyll-json-feeds).
- `jekyll-seo-tag` works fine. `lang` must be set for every collection (using front matter defaults), or else everything will have `og:locale` set to `en-US` (or whatever `site.lang` is set to). Doesn’t create `rel="alternate" hreflang="x"` or `og:locale:alternate` tags, but you can add those yourself.
