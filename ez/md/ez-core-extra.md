# Increase eZ power with EzCoreExtraBundle

By [Jérôme Vieilledent](https://github.com/lolautruche) / [@jvieilledent](http://twitter.com/jvieilledent)

http://lolautruche.github.io/ez/ezcoreextra.html

---

## What is EzCoreExtraBundle?

Set of additional tools to make eZ development smoother (and cooler) <!-- .element: class="fragment" -->

Designed for eZ Publish 5.4+ / eZ Platform 1.x <!-- .element: class="fragment" -->

Yes, it'll work with 2014.11 + Netgen variant :-) <!-- .element: class="fragment" -->

Symfony 2.7+ (2.8+ recommended) <!-- .element: class="fragment" -->

---

## What's in there?

* Configurable template variable injection <!-- .element: class="fragment" -->
* Context aware Twig global variables <!-- .element: class="fragment" -->
* Simplified authorization checks <!-- .element: class="fragment" -->
* Themes <!-- .element: class="fragment" -->

---

## Configurable template variable injection
Meant for simple to intermediate needs. 

Goal is to expose additional variables in your view templates from view template configuration.

Note: 
One often need to inject additional variables in view templates.

eZ offers possibility to implement custom controllers to *enrich* views. Very useful, but most of the time overkill when
you have simple needs.

--

## Different types of parameters
* Plain parameters, which values are directly defined in the configuration
* Container parameters: `%my.parameter%`
* Dynamic settings: <br>`$<paramName>[;<namespace>[;<scope>]]$`
* Parameter provider services

--

## Full example
```yaml
ezpublish:
    system:
        my_siteaccess:
            location_view:
                full:
                    article_test:
                        template: "@ezdesign/full/article_test.html.twig"
                        params:
                            osTypes: [osx, linux, losedows]
                            secret: '%secret%'
                            # Parameters resolved by config resolver
                            # Supported syntax for parameters: $<paramName>[;<namespace>[;<scope>]]$
                            # e.g. full syntax: $my_setting;custom_namespace;my_siteaccess$
                            # See https://doc.ez.no/display/EZP/Dynamic+settings+injection
                            languages_fallback: '$languages$'
                            another_variable: '$my.super_variable;acme$'
                            my_provider: {"provider": "my_param_provider"}
                        match:
                            Identifier\ContentType: article
```

Note:
All configured parameters are only available in the template spotted in the template selection rule.
No, we can't inject services and we'll never be able too!

--

## Template

```jinja
{% extends "@ezdesign/pagelayout.html.twig" %}

{% block content %}
    <h1>{{ ez_render_field(content, 'title') }}</h1>
    
    <p><strong>Secret:</strong> {{ secret }}</p>
    
    <p><strong>OS Types:</strong></p>
    {% for os in osTypes %}
        {{ os }}
        {% if not loop.last %}, {% endif %}
    {% endfor %}
{% endblock %}
```

--

## Parameter provider
Service called to dynamically inject variables in views.

* Lets you call your own dependencies
* Defines reusable sets of variables
* As flexible as a service can be!

--

## Parameter provider service definition
```yaml
services:
    acme_test.my_provider:
        class: Acme\TestBundle\MyViewParameterProvider
        arguments: ["@some_service"]
        tags:
            # alias must match with value configured under "provider" key in ezpublish.yml
            - {name: "ez_core_extra.view_parameter_provider", alias: "my_param_provider"}
```

--

## Parameter provider example
```php
use Lolautruche\EzCoreExtraBundle\Templating\ViewParameterProviderInterface;

class MyViewParameterProvider implements ViewParameterProviderInterface
{
    private $someService;

    public function getParameters(array $viewConfig)
    {
        // Current location and content are available in content/location views
        $location = $viewConfig['parameters']['location'];
        $content = $viewConfig['parameters']['content'];

        return array(
            'foo' => $this->someService->giveMeFoo(),
            'some' => 'thing'
        );
    }
}
```

--

## Template using parameter provider
```jinja
{% extends "@ezdesign/pagelayout.html.twig" %}

{% block content %}
    <h1>{{ ez_render_field(content, 'title') }}</h1>
    
    {# Param provider is namespaced by "my_provider" according to configuration #}
    <p>{{ my_provider.foo }}</p>
    <p>{{ my_provider.some }}</p>
{% endblock %}
```

---

## Context aware Twig global variables
Just like regular Twig global variables, but SiteAccess aware :-)

<br>
Useful for meta tags default values (title, description...) <!-- .element: class="fragment" data-fragment-index="1" -->
<br>or any variables you need available in any template. <!-- .element: class="fragment" data-fragment-index="1" -->

Note:
By default, Twig allows to inject global variables for the whole application.
When working on multisite instances, you may want to define variables that will be available only in the current SiteAccess.

--

### Global variables configuration

```yaml
ez_core_extra:
    system:
        my_siteaccess:
            twig_globals:
                my_variable: foo
                another_variable: 123
                something_else: [bar, true, false]
```

### Resulting template

```jinja
My variable: {{ my_variable }}<br>
Number variable : {{ another_variable }}<br>
<br>
{% for val in something_else %}
    {{ val }}<br>
{% endfor %}
```

---

## Simplified authorization checks

Note:
This feature simplifies the way you check authorization with eZ inner ACL system, 
using `module/function` and optionnaly a value object (e.g. a content object).

--

## Without EzCoreExtra
```php
use eZ\Bundle\EzPublishCoreBundle\Controller;
use eZ\Publish\Core\MVC\Symfony\Security\Authorization\Attribute as AuthorizationAttribute;

class MyController extends Controller
{
    public function fooAction()
    {
        // ...
        $accessGranted = $this->isGranted(
            new AuthorizationAttribute('content', 'read')
        );

        // Or with an actual content
        $accessGranted = $this->isGranted(new AuthorizationAttribute(
            'content', 'read', 
            ['valueObject' => $myContent]
        ));
    }
}
```

Not usable within a Twig template using <!-- .element: class="fragment" data-fragment-index="1" -->
`is_granted()` <!-- .element: class="fragment" data-fragment-index="1" -->

--

## With EzCoreExtra

`ez:<module>:<function>`

<br>
No more `AuthorizationAttribute`!

--

```php
use eZ\Bundle\EzPublishCoreBundle\Controller;

class MyController extends Controller
{
    public function fooAction()
    {
        // ...
        $accessGranted = $this->isGranted('ez:content:read');

        // Or with an actual content
        $accessGranted = $this->isGranted('ez:content:read', $myContent);
    }
}
```

--

## Twig friendly

```jinja
{% set accessGranted = is_granted("ez:content:read") %}

{# Or with an actual content #}
{% set accessGranted = is_granted("ez:content:read", my_content) %}
```

---

# Themes

--

## Themes
* Automatic fallback for templates and assets <!-- .element: class="fragment" -->
* Similar to legacy design fallback system <!-- .element: class="fragment" -->
* Not exportable (yet) <!-- .element: class="fragment" -->

--

## Terminology
* **Theme**: Labeled collection of templates/assets<br>
* **Design**: Labeled collection of themes, with fallback order.

Note:
Theme: Typically a directory containing templates<br>
Design: One design can be used by SiteAccess

--

## Design configuration
0. Declare your design, with a name and an ordered collection of themes to use
0. Configure it for your SiteAccess

--

## Design configuration
```yaml
ez_core_extra:
    design:
        # You declare every available designs under "list".
        list:
            # my_design will be composed of "theme1" and "theme2"
            # "theme1" will be the first tried. 
            # If template/asset cannot be found in "theme1", "theme2" will be tried out.
            my_design: [theme1, theme2]
    system:
        my_siteaccess:
            # my_siteaccess will use "my_design"
            design: my_design
```

--

## Templates
Theme directory must be located under<br><br>
`<bundle_directory>/Resources/views/themes/`<br><br>
and/or<br><br>
`app/Resources/views/themes/`

--

## Typical paths
* `app/Resources/views/themes/foo/` => `foo` theme.
* `app/Resources/views/themes/bar/` => `bar` theme.
* `src/AppBundle/Resources/views/themes/foo/` => `foo` theme.
* `src/FooBundle/Resources/views/themes/the_best/` => `the_best` theme.

--

## Themes for templates usage
Introducing `@ezdesign` Twig namespace!

```jinja
{# Will load 'some_template.html.twig' #}
{# directly under one of the specified themes directories #}
{{ include("@ezdesign/some_template.html.twig") }}

{# Will load 'another_template.html.twig', located under 'full/' directory #} 
{# which is located under one of the specified themes directories #}
{{ include("@ezdesign/full/another_template.html.twig") }}
```

Note:
Under the hood, theming system uses Twig namespaces. 
As such, Twig is the only supported template engine.

--

## Themes in template rules
```yaml
ezpublish:
    system:
        my_siteaccess:
            content_view:
                full:
                    home:
                        template: "@ezdesign/full/home.html.twig"
```

--

## Templates fallback order

* Global view override:<br>
  `app/Resources/views/`
* Global theme override:<br>
  `app/Resources/views/themes/<theme_name>/`
* Bundle theme directory:<br>
  `src/<bundle_directory>/Resources/views/themes/<theme_name>/`

Note:
The bundle fallback order is the instantiation order in `AppKernel`

--

## Additional override paths
```yaml
ez_core_extra:
    design:
        override_paths:
            - "%kernel.root_dir%/another_override_directory"
            - "/some/other/directory"
```

--

## Assets
Introducing `ezdesign` asset package!

```jinja
{{ asset("js/foo.js", "ezdesign") }}
{{ asset("js/foo.css", "ezdesign") }}
{{ asset("images/foo.png", "ezdesign") }}
```

--

## Typical paths for assets
* `<bundle_directory>/Resources/public/themes/foo/` => foo theme.
* `<bundle_directory>/Resources/public/themes/bar/` => bar theme.
* `web/assets/` => global override

--

## Assets fallback order
* Global assets override: `web/assets/`
* Bundle theme directory: `web/bundles/<bundle_directory>/themes/<theme_name>/`

<br>
Don't forget to run `assets:install`!

---

![That's all folks!](/assets/thats-all-folks.jpg)
