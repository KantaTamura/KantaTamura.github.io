## KantaTamura Portfolio Site

using [zola](https://www.getzola.org/), one of the SSGs, and the template used is [serene](https://github.com/isunjn/serene).

### deploy

```
$ zola serve
```

### update template

```
# update serene repo
$ cd themes/serene
$ git fetch upstream
$ git merge upstream/main

# update template file
$ cp -r templates/_custom_css.html ../../templates/_custom_css.html
$ cp -r highlight_themes ../..
$ cp -r static/icon ../../static
```

> [!NOTE]
> edit content width (768px to 1012px)
> ```
> --homepage-max-width: 1012px;
> --main-max-width: 1012px;
> ```
> `templates/_custom_css.html`
