# jekyll-vanilla-i18n

Turning Jekyll multilingual is not a straightforward task and comes with tradeoffs. This approach trades complex setup and configuration for simple content management.

Minimum required Jekyll version is 3.7.0, or 3.8.0 if using `collections_dir`.



## Configuration

Locales are configured in `_config.yml`:

```yaml
locales:
  default:
    baseurl: ""
    lang: en-US
    name: English
  pt:
    baseurl: /pt
    lang: pt-PT
    name: Português
```

The `default` locale is special, assumed to be the primary one and to be output to the site root.

### Collections

Collections are mirrored for each locale. Collections in the `default` locale are named normally, while other locales take the locale’s label as a suffix:

```yaml
collections:
  photos:
    output: true
    permalink: /photos/:path/
  photos_pt:
    output: true
    permalink: /pt/fotografias/:path/
```

Permalinks need to be set manually for each collection to match the locale’s `baseurl`.

### Front matter defaults

A few front matter defaults need to be configured (`*` glob patterns are helpful here).

To match localized collections to each other, `collection_basename` is used. All `photos` collections across any locales must have `collection_basename` set to `photos`:

```yaml
defaults:
- scope:
    path: "_photos*/"
  values:
    collection_basename: photos
```

And to associate collections with their respective locales, some more defaults are needed:

```yaml
- scope:
    path: "*_pt/"
  values:
    locale: pt
    lang: pt-PT
    collection_suffix: _pt
- scope:
    path: ""
  values:
    locale: default
    lang: en-US
```

Setting `lang` in this way allows plugins like jekyll-seo-tag to pick it up. `collection_suffix` is used in conjunction with `collection_basename` to match collections between locales. Documents in the `default` locale don’t have a `collection_suffix`.

For pages, `pages_LOCALE` collections need to be created as well. The default locale’s pages can be moved to a `pages` collection or left as normal pages. Because collections don’t quite act the same way as regular pages, permalinks for `index` documents won’t work as expected; when using `pretty` permalinks they will output to `/index/index.html`. This can only be avoided by setting `permalink` manually in front matter. This can be done in the documents themselves, or through more defaults:

```yaml
- scope:
    path: "_pages_pt/index.*"
  values:
    permalink: "/pt/"
```

## Content management

The site’s content should look something like this:

```
├── about.markdown
├── blog.markdown
├── index.markdown
├── photos.markdown
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

To localize permalinks, different filenames can be set for each locale. We’ll add front matter to match them up later.

```
├── _photos
│   ├── photo-1.markdown
│   └── photo-2.markdown
├── _photos_pt
│   ├── fotografia-1.markdown
│   └── fotografia-2.markdown
```

Collection labels themselves are never localized.

It’s not necessary for each locale to have a copy of every collection, or every file — content can be asymmetric.



### Document matching between locales

Document matching is automatic when file and folder names are exactly the same. In this example both documents represent the same content:

```
├── _collection
│   └── folder
│       └── document.markdown
└── _collection_pt
    └── folder
        └── document.markdown
```

To match them, the `i18n` include generates a variable called `document_id`, based on each document’s path inside the collection, and excluding the file extension.

Both documents in this example would return the same `document_id`, so they would match automatically:

```
folder/document
```

When filenames don’t match, `document_id` can be set manually via YAML front matter. Example:

```
├── _collection
│   └── folder
│       └── document.markdown
└── _collection_pt
    └── pasta
        └── documento.markdown
```

By adding this front matter to `documento.markdown`, they’d match:

```yaml
---
document_id: folder/document
---
```

This is better than using matching filenames and setting URL localizations through `page.permalink` because it’s derived from the document’s path in the site source, rather than the compiled site. That means the global permalink style can be changed without requiring changes to documents.



## Working with i18n in Liquid

The `i18n` include does most of the heavy lifting for making i18n manageable in Liquid. It defines the following variables:

| Variable | Description |
|:--|:--|
| `locale` | Contains the attributes of the current locale, as defined in `_config.yml`. The same as `site.locales[page.locale]`. |
| `localized_collections` | An array of labels for all collections matched through `page.collection_basename`. |
| `localized_pages` | An array of objects for all documents matched through `document_id`, based on `localized_collections`. |
| `default_page` | The page object of a matched `default` locale version of the current document. Useful for falling back to the default locale when dealing with unlocalized content. The same as `{{ localized_pages \| where: "locale", "default" \| first }}`. |
| `strings` | Contains any localized text strings defined in `_data`. More on that below. |
| `document_id` | Returns `page.document_id` or automatically generates a `document_id` string for the page. |

`i18n` must be included at the top of every layout requiring i18n features:

```liquid
{% include i18n/i18n %}
```

When it’s useful to get i18n variables for a document other than the current one, the `obj` parameter can be passed to the include. In this case, all variables will be prefixed with `obj_`, to avoid overwriting the current page’s variables. Example:

```liquid
{% include i18n/i18n %}

<h2>Collection Document IDs</h2>
{% for document in collection %}
{% include i18n/i18n obj=document %}
<p>{{ obj_document_id }}</p>
{% endfor %}

<h2>Current Page ID</h2>
<p>{{ document_id }}</p>
```

Since getting `document_id` is the most common use case, a smaller include that only assigns that variable can be used:

```liquid
{% include i18n/document_id %}
<p>{{ document_id }}</p> <!-- refers to {{ page }} -->

{% assign document = site.collection | first %}
{% include i18n/document_id obj=document %}
<p>{{ obj_document_id }}</p> <!-- refers to {{ document }} -->
```



### Content fallbacks

`default_page` can be used to fallback to content in the default locale if it’s not localized:

```yaml
{{ page.image | default: default_page.image }}
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
document_id: about
---
Somos uma empresa fixe.
```



### Localizing text strings

Text strings are defined in `_data` — one YAML file for each locale, named `strings_LOCALE.yml`. The `default` locale file is named `strings.yml`.

```
└── _data
    ├── strings_pt.yml
    └── strings.yml
```

Any keys set in these files can be accessed directly through the `strings` variable:

```liquid
{{ page.locale }}: {{ strings.hello }}
```

```
default: Hello
pt: Olá
```



### Localizing dates

A special include deals with dates: `i18n/date`.

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

```liquid
{% include i18n/date date=page.date format="long" %}
```

```
default: September 15, 2016
pt: 15 Setembro 2016
```



## Plugin compatibility

I’ve tested the Jekyll plug-in trinity with this site:

- `jekyll-sitemap` includes every page without a problem, but they aren’t marked up using [rel="alternate" links](https://support.google.com/webmasters/answer/2620865?hl=en).
- `jekyll-feed` will only generate a feed for the default `posts` collection. To get localized feeds, liquid layouts like [jekyll-rss-feeds](https://github.com/snaptortoise/jekyll-rss-feeds) can be used.
- `jekyll-seo-tag` works fine. `lang` must be set for every collection (using front matter defaults), or else everything will have `og:locale` set to `en-US` (or whatever `site.lang` is set to). Doesn’t create `rel="alternate" hreflang="x"` or `og:locale:alternate` tags, but those can be added manually.
