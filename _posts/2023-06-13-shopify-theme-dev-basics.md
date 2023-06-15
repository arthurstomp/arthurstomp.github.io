---
layout: post
tags: [shopify, front-end]
title: Shopify Theme Dev Basics
---

> All code of this post is available at [https://github.com/arthurstomp/shopity_theme_basics](https://github.com/arthurstomp/shopity_theme_basics)

The shopify theme development workflow may feel kinda weird at first.
But once get hold of it you will enjoy how practical it is.

### Where, in the Shopify store, the themes lives.

![Where themes live 1][where1_highlight]

![Where themes live 2][where2]

[where1_highlight]: /assets/images/2023-06-13-shopify-theme-dev-basics/where1_highlight-online-store.png
[where2]: /assets/images/2023-06-13-shopify-theme-dev-basics/where2.png

We will be fully customizing the theme.
So it's better to keep track of the code.
Go on "Download theme file" and init git in it.

### Running it "locally" - Shopify CLI

The entire shopify theme development workflow revolves around the [`Shopify CLI`][shopify_cli].
It will allow you to start the development server, create "unpublished themes" with their own preview and editor, lint the theme, etc.

To get started, go to the folder where you downloaded the "Theme files" and execute `$ shopify theme dev --store="<your-store-subdomain>"`. This will prompt you to sign into the store. Once you sign in, the cli will start the local server with the local theme, sync your local files with a hidden online dev-theme, providing link for both the dev-theme online preview and editor.
By default, the sync of local -> online dev-theme is just one way, changes in local files are reflected to the online dev-theme.

If everything goes well you should be able to access you development theme at [http://localhost:9292](http://localhost:9292).

> If you are working on the same store, once you sign-in to a store with `shopify theme dev`, you can omit the `--store`. Otherwise you will need o execute `$ shopify auth logout` and explicitly sign-in to the new store with `$ shopify theme dev --store="<the other store subdomain>"`.

> You can set theme:dev to sync both ways(local <-> online dev-theme) by using `--theme-editor-sync` option - `$ shopify theme dev --theme-editor`.

> I, personally, don't like to use the full sync. When i have changes done in the online editor i just execute `$ shopify theme pull -d` to pull the changes from the online editor to your local files.

[shopify_cli]: https://www.npmjs.com/package/@shopify/cli

### Folder structure

```
.
├── assets
├── config
├── layout
├── locales
├── sections
├── snippets
└── templates
```

A shopify theme is just a collection of `templates` that are made available to Shopify dashboard to assign to its products and pages.
Those templates are implemented as a group of sections, dynamic or static, and snippets.
Sections are the main component of building a template it define `settings` and `blocks` to be customized through the theme editor.
Snippets are just snippets, parts of code to be reused.

- `assets`: Holds `js` and `css` files to be referenced on layout, template and sections.
- `config`: Holds 2 important files: `settings_data.json` and `settings_schema.json`. `settings_schema.json` defines the schema expected to be declared at `settings_data.json` - See [Settings Schema docs][liquid_settings_obj] to understand its format, and [Shopify documentation][shopify_settings_schema_doc] for more context.
- `layout`: Holds layout liquid files to be used on `templates` schemas - See [Shopify documentation][shopify_layout_doc] for more context.
- `locales`: Holds `json` files containing translation of editor components into a variaty of languages.
- `sections`: Holds section `liquid` files.
- `snippets`: Holds snippets `liquid` files.
- `templates`: Holds templates `json` and `liquid` files.

[liquid_settings_obj]: https://shopify.dev/docs/api/liquid/objects/settings
[shopify_settings_schema_doc]: https://shopify.dev/docs/themes/architecture/settings#settings_schema-json
[shopify_layout_doc]: https://shopify.dev/docs/themes/architecture/layouts

### Creating a new page - Template

Shopify themes require some basic templates to be present. Those are the ones present at the `dawn theme`.

Templates names are important they define their use by shopify editor. Because of that, new custom templates need to have those names as prefix. For example, to have a custom product page for xmas create a new file call `templates/product.xmas.json`

> See [Shopify Template documentation][shopify_template_doc] for more information.

Once the file is created, **and it complies to theme shopify template format**, this new template will be available at you dev-theme editor.

![Shopify template placement 1][template_placement1]

![Shopify template placement 2][template_placement2]

Once published the new template, in the case of product, will appear at the bottom of your product page allowing you to set the page of that product for the custom template.

![Product template assignement 1][template_product_template_assing1]

[shopify_template_doc]: https://shopify.dev/docs/themes/architecture/templates
[template_placement1]: /assets/images/2023-06-13-shopify-theme-dev-basics/template_placement1-highlight.png
[template_placement2]: /assets/images/2023-06-13-shopify-theme-dev-basics/template_placement2-highlight.png
[template_product_template_assing1]: /assets/images/2023-06-13-shopify-theme-dev-basics/template_product_template_assing1-highlight.png

### Change in the editor pull to local

At this point `product.xmas.json` is no different from `product.json`. For a first customization, lets add a rich text section to it, edit its content and hit "save".

![Add rich text section to product.xmas.json][editor_change1]

Once saved, bring those change to local environment by executing `$ shopify theme pull -d`.

[editor_change1]: /assets/images/2023-06-13-shopify-theme-dev-basics/editor_change1-highlight.png

### New layout

JSON templates are basically the declaration of section settings and blocks, and their order. Those sections are rendered on the layout.
So, to change the looks of our new product page, lets create a new layout for it.

> Templates without an explicity layout default to `theme.liquid`.

To keep everything consistent, lets just copy `theme.liquid` into `xmas.liquid`. That will keep theme setting working.

After that, set the `product.xmas.json` layout to `xmas`.

```javascript
{
  "layout": "xmas",
  "sections": {
    "main": {
      "type": "main-product",
      "blocks": {
        "vendor": {
          "type": "text",
          "settings": {
            "text": "{{ product.vendor }}",
            "text_style": "uppercase"
          }
        },
    ...
}
```

To add some style change, and keep everything organized, lets create `assets/xmas.css` and reference it at the bottom of `xmas.liquid` `<head>`

![Add xmas.css to xmas layout][layout_add_xmas-css]

*assets/xmas.css*

```css
.header-wrapper {
  background-color: red;
}

.header__heading-link:hover .h2 {
  color: lightgray;
}

.header__heading-link .h2 {
  color: white;
}

.header__icon {
  color: white;
}

.header__menu-item {
  color: white;
}

.header__menu-item:hover {
  color: lightgray;
}
```

![layout_edited][layout_edited]

[layout_add_xmas-css]: /assets/images/2023-06-13-shopify-theme-dev-basics/layout_add_xmas-css.png
[layout_edited]: /assets/images/2023-06-13-shopify-theme-dev-basics/layout_edited.png

### Customizable image on Product Info section

Copy `sections/main-product.liquid` into `sections/xmas-product.liquid`.

Change its `schema.name` and `schame.presets.name` to `Xmas Product info`.

```javascript
{
  "name": "Xmas Product Info",
  "tag": "section",
  "class": "section",
  "blocks": [
    ...
  ],
  "settings": [
    ...,
    {
      "type": "image_picker",
      "id": "sides_background_image",
      "label": "Sides Background Image"
    }
  ],
  "presets": [
    {
      "name": "Xmas Product Info"
    }
  ]
}
```

> Sections without a `presets` **will not** appear on the editor

This new section will appear as available on the "+ Add Section" menu with the name of "Xmas Product Info".

![product_info_add_section][product_info_add_section]

You can replace the "main-product" by "xmas-product" on the editor and pull the changes. Or just update the `templates/product.xmas.json` changing `"type": "main-product",` by `"type": "xmas-product",`.

Note on the right side, while highlighting "Xmas Product Info" section, that there is a new setting on it, "Sides background image". That is the input for `\{\{sections.settings.sides_background_image\}\}` object. We will be using it to customize the section.

*xmas-product.liquid*

```html
\{\% style \%\}
.xmas-background-img--{{ section.id }} {
  background: url("{{section.settings.sides_background_image | image_url: width: 100}}");
}

.xmas-background-img--{{ section.id }} .product {
  background: white;
}
\{\% endstyle \%\}
<section
  id="MainProduct-\{\{ section.id \}\}"
  class="page-width section-\{\{ section.id \}\}-padding xmas-background-img--\{\{ section.id \}\}">
  ...
</section>
```

![product_info_customized][product_info_customized]

[product_info_add_section]: /assets/images/2023-06-13-shopify-theme-dev-basics/product_info_add_section.png
[product_info_customized]: /assets/images/2023-06-13-shopify-theme-dev-basics/product_info_customized.png

### Gathering Feedback

Shopify theme infrastructure is very helpful for gathering feedback of stackholders. To have and online preview of changes on a theme without using your online theme preview you can push a theme to the shopify so other users can preview it, customize it; and at the end publish it.

To push the current state of the theme, execute `$ shopify theme push -u --store="<your-shopify-store-id>" --theme="<new-theme-name>"`.

> `-u` option will create a new theme on the theme library. After the first push on a `--theme` you can omit the `-u` option.

![feedback][feedback]

[feedback]: /assets/images/2023-06-13-shopify-theme-dev-basics/feedback.png
