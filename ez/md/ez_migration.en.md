# Demystify migration to eZ 5.x

By [Jérôme Vieilledent](https://github.com/lolautruche) / [@jvieilledent](http://twitter.com/jvieilledent)

http://share.ez.no/blogs/core-development-team

http://lolautruche.github.io/ez/ez-migration.html

---

# Who are the mad (wo)men?

---

## (Not so) Good reasons for not migrating
* "I'm productive enough with legacy"
* "I'll need to learn everything from scratch!"
* "Symfony is too complicated for me"

---

## Why you really should migrate
* Performance
* Easier to develop with (and I mean it!)
* Future proof
* ~~Because it's cool~~

---

# No, you don't have to migrate everything at once!

---

## Easy steps for an eZ migration
0. Don't touch your content <!-- .element: class="fragment" -->
0. Don't touch your content templates <!-- .element: class="fragment" -->
0. Let the BC magic occur <!-- .element: class="fragment" -->

![Backwards compatibility motto](/ez/assets/bc_motto.jpg) <!-- .element: class="fragment" -->

---

# eZ 5 is designed for step-by-step migrations

---

# A few concepts to grasp first...

--

## Twig and template inheritance
* No more "2 steps view" <!-- .element: class="fragment" -->

* Templates will inherit from the main layout and enrich it <!-- .element: class="fragment" -->

* Think templates like PHP classes, with properties you can override <!-- .element: class="fragment" -->

* Those properties are named "blocks" <!-- .element: class="fragment" -->

--

## HMVC
* Stands for Hierarchical Model View Controller <!-- .element: class="fragment" -->

* No more fetch functions, instead you'll call controllers with sub-requests <!-- .element: class="fragment" -->

* Sub-requests can be inline, ESI or Hinclude <!-- .element: class="fragment" -->

--

## Dependency injection
* Everything is a *service* (think labelled class)

* Get your dependencies via the **Service Container**

* No more Singletons :-)

---

## First minimal steps
## 1. Base settings
```bash
php ezpublish/console ezpublish:configure <package_name> <admin_siteaccess_name>
```

--

## 1. Base settings

```bash
php ezpublish/console ezpublish:configure my_website my_admin_sitaccess
```

> By default, using a package name different than `ezdemo_site` or `ezdemo_site_clean` will enforce
> using `legacy_mode`.
>
> You can remove this setting later.

---

## 2. Create your bundle(s)

```bash
php ezpublish/console generate:bundle
```

---

## 3. Migrate your pagelayout

* Only requirement is to have a Twig block named "content"

* Split it up with **include** and **use blocks** so that you can override/enrich them in child-templates

* Use sub-requests and ESIs when you need to add more logic

--

## Settings for content view fallback
```yaml
ez_publish_legacy:
    system:
        my_siteaccess:
            templating:
                view_layout: AcmeDemoBundle::my_layout.html.twig
```

https://doc.ez.no/display/EZP/Legacy+template+fallback

--

## Use the same layout base for legacy modules
```yaml
ez_publish_legacy:
    system:
        my_siteaccess:
            templating:
                view_layout: AcmeDemoBundle::layout.html.twig
                module_layout: AcmeDemoBundle::layout_legacy.html.twig
```

```jinja
{% extends "AcmeDemoBundle::pagelayout.html.twig" %}

{% block content %}
    {# module_result variable is received from the legacy controller. #}
    {# It holds the legacy module result #}
    {{ module_result.content|raw }}
{% endblock %}
```

https://doc.ez.no/display/EZP/Legacy+template+fallback

--

## Include legacy templates
```jinja
{{ include("design:foo/my_legacy_template.tpl") }}

{{ include("file:/path/to/my_legacy_template.tpl") }}
```

---

# Metal France

![Metal France logo](/ez/assets/metalfrance_logo.png)

http://www.metalfrance.net

https://github.com/lolautruche/metalfrance

--

## Current website
* Current version number is v3.1
* eZ v2011.06 (~4.5)
* Apache 2.2 / PHP 5.4 + FPM / MySQL 5.1

--

## Plans: v3.5
* Migration to eZ 5.3 with minimum migration (layout + few content templates)
* Use of Varnish
* No migration of custom datatypes
* Still using eZ Find for search

--

## Plans: v4.0
* Full use of Platform stack (no legacy templates)
* Custom datatypes migrated (and open-sourced)
* Use of new Solr search engine

---

# Let's show some code!

https://github.com/lolautruche/metalfrance
