# Jekyll 4.0 i18n proposal

This is a hypothetical solution for native internationalization support in Jekyll 4.0.

This branch’s code represents how the master branch’s site would work with that solution in place. The differences can be compared in [this commit](https://github.com/letrastudio/jekyll-i18n/commit/e90895716a61c3929911f7f0f54f6ced96abcbad).


## Configuration

There would be no option to enable i18n. A Jekyll site with no configured locales would in fact have a single non-optional locale, labeled `default`. Additional locales and any extra attributes could be defined in `_config.yml`:

```yaml
locales:
  default:
    lang: en-US
    name: English
  pt:
    lang: pt-PT
    name: Português
```

With just this configuration, english content builds to `_site/`, and portuguese content builds to `_site/pt/`. We could override this behavior by setting a `baseurl` attribute for each locale.

### Collections

Collections would be automatically mirrored for each locale (creating a `photos` collection would make it exist for every locale), but locale-specific attributes could be set. For the portuguese locale in this example, permalinks could be translated like so:

```yaml
collections:
  photos:
    output: true
    permalink: /photos/:path
  photos/pt:
    permalink: /fotografias/:path
```

The `collection/locale` syntax is valid YAML, and would mirror folder structure (explained in the next section). The same syntax could be used for front matter defaults:

```yaml
defaults:
- scope:
    path: ""
    type: pages
  values:
    flag: 🇺🇸
- scope:
    path: ""
    type: pages/pt
  values:
    flag: 🇵🇹
```


## Content management

Locales wouldn’t inherit any content from `default`. Each document would have to be specifically added inside `_LOCALE` folders. In our example, that means creating `_pt` folders for each type of content:

```
├── _photos
│   ├── _pt
│   │   ├── fotografia-1.markdown
│   │   └── fotografia-2.markdown
│   ├── photo-1.markdown
│   └── photo-2.markdown
```

Having locale folder names start with `_` means they will simply be ignored if a corresponding locale is not present in `site.locales`.

Document and subfolder filenames don’t have to match (we’ll get to that in a second). It’s also not mandatory to add localized versions of everything — each locale can have different sets of content.

Building our site now should result in something like this:

```
_site
├── photos
│   ├── photo-1/index.html
│   └── photo-2/index.html
└── pt
    └── fotografias
        ├── fotografia-1/index.html
        └── fotografia-2/index.html
```



### Document matching between locales

A new document attribute called `document_id` would be used to establish matches between locales. It would be generated automatically for every document based on its path within the parent collection and filename, excluding the file extension. Some examples:

| Document | `document_id` |
|:--|:--|
| `index.html` | `index` |
| `_pt/index.html` | `index` |
| `_collection/folder/document.md` | `folder/document` |
| `_collection/_pt/pasta/documento.md` | `pasta/documento` |

Jekyll would use `document_id` to match files between locales, so our `index` pages would be matched automatically. To force a match between the other pair of documents, `document_id` could be overridden in `_collection/_pt/pasta/documento.md` using YAML front matter:

```
---
document_id: folder/document
---
```

This is preferable to using matching filenames and setting URL localizations through `page.permalink` because it’s derived from the document’s path in the site source, rather than the compiled site. That means the global permalink style can be changed without requiring changes to documents.


## Working with i18n in Liquid

Here’s where it gets interesting. This proposal adds a new global variable in Liquid: `{{ locale }}`.

It would include locale metadata defined in `_config.yml`:

| Liquid | Output (`default` locale) | Output (`pt` locale)
|:--|:--|:--|
| `locale.label` | `'default'` | `'pt'` |
| `locale.baseurl` | `''` | `'/pt'` |
| `locale.lang` | `'en'` | `'pt-PT'` |
| `locale.name` | `'English'` | `'Português'` |


More interestingly, it would also act as a counterpart to the `site` variable, containing its own versions of the site’s content:

```liquid
{{ locale.pages }}
{{ locale.posts }}
{{ locale.collections }}
{{ locale.photos }}
{{ locale.tags }}
{{ locale.data }}
...
```

Other `site` variables not related to content wouldn’t be brought over, (like `plugins`, `collections_dir`, `exclude`, etc.).

Using `site` would work just as it does now. `site.posts` would always return posts from the `default` locale. This distinction is important for dealing with content asymmetry between locales. For example, when building a blog index, you might want to loop through `site.posts`, linking to localized versions of posts when available, and falling back to `default` locale versions when not. Or you can loop through `locale.posts` and only show posts that exist in that locale.

`site.locales` would be exposed to Liquid as an array, like `site.collections`. To access a specific locale’s content, the `where` filter could be used:

```liquid
{% assign pt_locale = site.locales | where: "label", "pt" | first %}
{% for post in pt_locale.posts %}
  <li><a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
```


### Creating relationships between locales

The `{{ page }}` global variable would get some new attributes:

- `page.document_id`, the strings we’re using for matching documents
- `page.locale`, containing a string with the locale’s label (e.g. `'default'`, `'pt'`)
- `page.localized`, for accessing versions of the page in different locales, kind of like `page.previous` and `page.next`

That last one would contain a Liquid object for every page matched using `document_id`. You’d use it like this:

```liquid
{{ page.localized.LOCALE }}
```

It could be used to reference different localized versions of the same page:

```liquid
{% if page.localized.pt %}
  {% assign pt_locale = site.locales | where: "label", "pt" | first %}
  Read this page in Portuguese:
  <a href="{{ page.localized.pt.url }}" hreflang="{{ pt_locale.lang }}">{{ page.localized.pt.title }}</a>
{% endif %}
```


### Content fallbacks

`page.localized` could be used to fallback to content in a different locale if it’s not localized, using the `default` filter:

```liquid
{{ page.image | default: page.localized.default.image }}
```

In most cases this would be used to fallback to the `default` locale, so it would be nice to add `page.default` as an alias to `page.localized.default`:

```liquid
{{ page.image | default: page.default.image }}
```

It’s important that this sort of behavior be manually defined in Liquid, because falling back to the `default` locale may not always be desirable. For example, you might want to set up a cascade of fallbacks though various locales if more than one uses the same language:

```liquid
{% if page.locale == "fr_CA" %}
{{ page.headline | default: page.localized.fr_FR.headline | default: page.default.headline }}
{% endif %}
```

Using fallbacks means that each localized file only needs to contain localizable content. The `about.markdown` file in the `default` locale might look like this:

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

String localization could be solved using `_data`:

```
└── _data
    ├── _pt
    │   └── strings.yml
    └── strings.yml
```

And this syntax in Liquid:

```liquid
{{ locale.data.strings.hello }}
```

But with string localization being such a common use case, a special `data`-like `_strings` folder might be justified, which would accept a single YAML file per locale:

```
└── _strings
    ├── default.yml
    └── pt.yml
```

Not _really_ necessary, but it would standardize string localization and result in cleaner, easier to remember code:

```liquid
{{ locale.strings.hello }}
```

### Localizing dates

This might be the hardest i18n problem to solve well, and it may be feasible to not include some sort of new date filter on day one.

My solution would be very similar to what I’m using for Jekyll 3.

Locale-specific date formats would be set in `_strings/LOCALE.yml` files using the `date_formats` key:

```yaml
date_formats:
  full: "%A, %e %B %Y"
  long: "%B %e, %Y"
  short: "%m/%d/%Y"
```

Month and weekday translations could also be set:

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

A new `localized_date` filter would use those settings for formatting:

```liquid
{{ page.locale }}: {{ page.date | localized_date: "full" }}
```

```
default: Thursday, 15 September 2016
pt: Quinta-feira, 15 de Setembro de 2016
```

While it would work, it’s cumbersome to set up. Maybe it could recognize ISO locale codes and spit out a correctly localized string:

```liquid
{{ page.date | localized_date: "long", "zh-CN" }}
```

But this feels a bit beyond the scope of what Jekyll should do, and it would be unreasonable to assume that *every* date format would be covered.
