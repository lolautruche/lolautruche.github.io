# Étendre Symfony

### Les bonnes pratiques
Par [Jérôme Vieilledent](https://github.com/lolautruche) / [@jvieilledent](http://twitter.com/jvieilledent)

http://share.ez.no/blogs/core-development-team

http://lolautruche.github.io/ez/etendre-symfony.html



## Pourquoi étendre Symfony ?

> Et si je veux faire `<remplacer_par_un_truc_farfelu>` ?

Signé: Un chef de projet très imaginatif...

Note:
- Exemple eZ : router dynamique<br>
- Beaucoup pensent qu'un framework est une prison, ce qui est rarement le cas, en particulier pour Symfony<br>
- Symfony a été pensé pour être extensible.<br>
- Possible de modifier le comportement de Symfony, même à des endroits improbables, sans magie noire.<br>



## Qui peut étendre Symfony ?

Tout le monde ! <!-- .element: class="fragment" -->

Note:
- On peut aborder Sf à plusieurs niveaux de dev (basique, intermédiaire, avancé)<br>
- **Basique**: "utilisation" du framework et de bundles tiers (configuration, controllers, templates, commandes simples...)<br>
- **Intermédiaire**: Implémentation de services basés sur d'autres, on commence à penser extensibilité<br>
- **Avancé**: Agissement bas niveau<br>



# Astuce #1
## Configurer des bundles tiers <br>depuis le sien

Note:
Peut paraître anodin, mais incroyablement utile quand on maintient des bundles *génériques*.<br>



# 2 façons de faire
1. Explicite depuis le config.yml principal
2. Implicite depuis l'extension DIC



# Explicite
```yaml
# app/config/config.yml

imports:
    # On importe la configuration Assetic se situant dans notre bundle
    - {resource: "@AcmeTestBundle/Resources/config/assetic.yml"}
```

```yaml
# src/Acme/TestBundle/Resources/config/assetic.yml

assetic:
    # On ajoute simplement AcmeTestBundle aux bundles Assetic
    bundles: [AcmeTestBundle]
```



## Avantages
* Rapide <!-- .element: class="fragment" -->
* Explicite <!-- .element: class="fragment" -->
<br><br>

## Inconvénients <!-- .element: class="fragment" -->
- Obligation d'importer depuis la conf principale <!-- .element: class="fragment" -->
- Impossible de surcharger les valeurs <!-- .element: class="fragment" -->



## Implicite
```php
namespace Acme\TestBundle\DependencyInjection;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Extension\PrependExtensionInterface;
use Symfony\Component\Config\Resource\FileResource;
use Symfony\Component\HttpKernel\DependencyInjection\Extension;
use Symfony\Component\Yaml\Yaml;

class AcmeTestExtension extends Extension implements PrependExtensionInterface
{
    public function prepend( ContainerBuilder $container )
    {
        $file = __DIR__ . '/../Resources/config/assetic.yml';
        $config = Yaml::parse(file_get_contents($file));
        // On ajoute explicitement la configuration pour le namespace "assetic"
        $container->prependExtensionConfig( 'assetic', $config );
        // On ajoute la ressource pour tracker les modifications en debug
        $container->addResource(new FileResource($file));
    }
}
```

```yaml
# On ne précise pas le namespace car déjà spécifié dans AcmeTestExtension
bundles: [AcmeTestBundle]
```



## Avantages
* Automatique <!-- .element: class="fragment" -->
* Surcharge de paramètres possible depuis config.yml <!-- .element: class="fragment" -->
<br><br>

## Inconvénients <!-- .element: class="fragment" -->
* Implicite, donne sentiment de "magie" <!-- .element: class="fragment" -->



# Astuce #2
## Les service tags



## Étendre des services sans les modifier
* Cache clearers / Cache warmers
* Extensions Twig
* Security voters
* ...
<br><br>
http://symfony.com/doc/current/reference/dic_tags.html

Note:
- Généralement un service tag est accompagné d'une interface




## Exemple: Cache clearer
```yaml
parameters:
    acme.my_cache_clearer.class: Acme\TestBundle\Cache\CacheClearer

services:
    acme.my_cache_clearer:
        class: %acme.my_cache_clearer.class%
        tags:
            - { name: kernel.cache_clearer }
```

```php
namespace Acme\TestBundle\Cache;

use Symfony\Component\HttpKernel\CacheClearer\CacheClearerInterface;

class CacheClearer implements CacheClearerInterface
{
    public function clear($cacheDir)
    {
        // ...
    }
}
```

Note: À chaque fois que cache:clear sera appelé, ce cache clearer sera déclenché.



# Astuce #3
## Event listeners/subscribers



## Event listeners/subscribers
**LES** points d'extension *visibles*.

Très nombreux et permettent de déclencher des actions et/ou modifier certains comportements.

> Il y a toujours un événement pour ça !



## Quelques événements
`Symfony\Component\HttpKernel\KernelEvents`

* kernel.request <!-- .element: class="fragment grow" -->
* kernel.controller <!-- .element: class="fragment grow" -->
* kernel.view <!-- .element: class="fragment grow" -->
* kernel.response <!-- .element: class="fragment grow" -->
* kernel.exception <!-- .element: class="fragment grow" -->
* kernel.terminate <!-- .element: class="fragment grow" -->


`Symfony\Component\Security\Http\SecurityEvents`<br>
`Symfony\Component\Security\Core\AuthenticationEvents`

* security.interactive_login <!-- .element: class="fragment grow" -->
* security.switch_user <!-- .element: class="fragment grow" -->
* security.authentication.success <!-- .element: class="fragment grow" -->
* security.authentication.failure <!-- .element: class="fragment grow" -->


`Symfony\Component\Console\ConsoleEvents`

* console.command
* console.terminate
* console.exception
<br><br>
`Symfony\Component\Form\FormEvents`

* form.pre_bind
* form.bind
* form.post_bind
* form.pre_set_data
* form.post_set_data




## Exemple d'un subscriber
```yaml
parameters:
    acme.my_subscriber.class: Acme\TestBundle\EventListener\MySubscriber

services:
    acme.my_subscriber:
        class: %acme.my_subscriber.class%
        arguments: [@logger]
        tags:
            - { name: kernel.event_subscriber }
```

```php
class MySubscriber implements EventSubscriberInterface
{
    private $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public static function getSubscribedEvents()
    {
        return array(
            KernelEvents::REQUEST => array('onKernelRequest' => 5)
        );
    }

    public function onKernelRequest(GetResponseEvent $event)
    {
        if (HttpKernelInterface::MASTER_REQUEST !== $event->getRequestType()) {
            return;
        }

        $this->logger->notice("Wow, j'ai fait un request listener !");
    }
}
```

Note:
- Subscriber mieux car on maintient les événements souscrits dans la classe
- Attention à l'instanciation des listeners/subscribers avec les dépendances, surtout quand kernel.request (très tôt)
- Kernel events => Vérifier le requestType (attention aux sub-requests)



## Debugger d'events
https://github.com/egulias/ListenersDebugCommandBundle

"egulias/listeners-debug-command-bundle"



# Astuce #4
## Étendre les services Symfony
### (sans tout casser) <!-- .element: class="fragment" -->



## #4.1 Héritage simple
* Parfait pour les besoins *simples*
* Un simple paramètre à modifier


## #4.1 Héritage simple
```yaml
# src/Acme/TestBundle/Resources/config/services.yml
parameters:
    router.class: Acme\TestBundle\Routing\DefaultRouter
```

```php
namespace Acme\TestBundle\Routing;

use Symfony\Bundle\FrameworkBundle\Routing\Router;
use Symfony\Component\Routing\Matcher\RequestMatcherInterface;
use Symfony\Component\HttpFoundation\Request;

class DefaultRouter extends Router implements RequestMatcherInterface
{
    public function matchRequest( Request $request )
    {
        // ...
    }
}
```



## #4.2 Et mes dépendances ?
Mantra à répéter chaque jour : <!-- .element: class="fragment" data-fragment-index="1" -->

"Je ne redéfinis pas les services des autres..."  (Amen) <!-- .element: class="fragment" data-fragment-index="1" -->

Note:
- Redéfinir des services ouvre la porte aux régressions (modifications des dépendances)
- Difficile à maintenir
- Bombe à retardement


## #4.2 Les compiler passes
* Éxécutées à la compilation du service container
* Permettent d'accéder à *tous* les services définis
* Permettent de modifier n'importe quel service défini

http://symfony.com/doc/current/cookbook/service_container/compiler_passes.html


### Exemple
```yaml
parameters:
    # Redefining the default router class to implement the RequestMatcherInterface
    router.class: eZ\Bundle\EzPublishCoreBundle\Routing\DefaultRouter
```

```php
class DefaultRouter extends Router implements RequestMatcherInterface
{
    public function matchRequest( Request $request )
    {
        // ...
    }

    public function setConfigResolver( ConfigResolverInterface $configResolver )
    {
        $this->configResolver = $configResolver;
    }
}
```


### La compiler pass
```php
class ChainRoutingPass implements CompilerPassInterface
{
    /**
     * @param \Symfony\Component\DependencyInjection\ContainerBuilder $container
     */
    public function process( ContainerBuilder $container )
    {
        if ( !$container->hasDefinition( 'router.default' ) )
            return;

        $defaultRouter = $container->getDefinition( 'router.default' );
        $defaultRouter->addMethodCall(
            'setConfigResolver',
            array( new Reference( 'ezpublish.config.resolver' ) )
        );
    }
}
```


### Enregistrement dans le bundle
```php
namespace eZ\Bundle\EzPublishCoreBundle;

use eZ\Bundle\EzPublishCoreBundle\DependencyInjection\Compiler\ChainRoutingPass;

class EzPublishCoreBundle extends Bundle
{
    public function build( ContainerBuilder $container )
    {
        parent::build( $container );
        $container->addCompilerPass( new ChainRoutingPass );
    }
}
```


### Ajouter une factory
```php
class FragmentPass implements CompilerPassInterface
{
    public function process( ContainerBuilder $container )
    {
        if ( !$container->hasDefinition( 'fragment.listener' ) )
        {
            return;
        }

        $fragmentListenerDef = $container->findDefinition( 'fragment.listener' );
        $fragmentListenerDef
            ->setFactoryService( 'ezpublish.fragment_listener.factory' )
            ->setFactoryMethod( 'buildFragmentListener' )
            ->addArgument( '%fragment.listener.class%' );
    }
}
```



## 4.3 Les alias
Permettent de substituer un service par un autre.

... Ou comment redéfinir un service, sans le redéfinir ! <!-- .element: class="fragment" -->



## 4.3 Les alias
Possible depuis une CompilerPass ou la classe DIC extension :
```php
$containerBuilder->setAlias( 'router', 'ezpublish.chain_router' );
```



# Astuce #5
## Rendre son code extensible

Suivre les mêmes pratiques que Symfony :

 * Classes paramétrées
 * Utiliser des événements
 * Service tags
 * Aliases

Note:
- Penser extensibilité lorsque l'on code. Se mettre à la place du développeur qui va utiliser sa lib / son bundle
- Réfléchir au private vs protected
- Découpler au maximum pour pouvoir réutiliser son code par la suite



# Fin
## ?  <!-- .element: class="fragment" -->

http://share.ez.no/blogs/core-development-team
