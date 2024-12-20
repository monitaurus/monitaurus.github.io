# How to Create a Behat Extension

> ℹ️ You can now found an edited version of this post into Behat documentation's cookbooks: [Creating a Behat extension to provide custom configuration for Contexts](https://docs.behat.org/en/latest/cookbooks/creating_a_context_configuration_extension.html).

I like using [Behat](https://docs.behat.org/en/latest/#) as it brings user-oriented concerns within the hands of developers. It's also a mature tool, simply bridging the gap between the [Gherkin syntax](https://cucumber.io/docs/gherkin/reference/) and PHP. No more needed. No more new feature to add.

While looking to automatically validate API's response against an [OpenAPI](https://spec.openapis.org/oas/latest.html) schema, I found out about Behat extensions. Plenty are available on [GitHub](https://github.com/search?o=desc&q=behat+extension+in%3Aname%2Cdescription+language%3APHP&ref=searchresults&s=stars&type=Repositories&utf8=%E2%9C%93). But strangely enough, I didn't find any documentation within Behat's site. I had to look into Behat and extension's code to understand what they can do, and how to create one.

## Why Create a Behat Extension?

If your sole aim is to make steps and their contexts easily accessible, extensions might not be the best fit. A more suitable approach would be to host those files somewhere available for download.

Extensions truly shine when configuration becomes a necessity.

In this tutorial, we'll craft a simple extension named `LogsExtension` that logs scenario duration.

Our journey will revolve around three key concepts:
- Constructing a custom configuration tree.
- Integrating this configuration into Behat's container.
- Initializing contexts with these values.

## Setting Up the Context

First, we need to create a context that will handle the logging. The `LogsExtension` will provide a `LogsContext` that hooks into scenarios to log their start and end times.

```
src/
    Context/
        LogsContext.php <--- This is where we'll implement our logging logic.
```

```php
namespace Behat\LogsExtension\Context;

use Behat\Behat\Context\Context;
use Behat\Behat\Hook\Scope\BeforeScenarioScope;
use Behat\Behat\Hook\Scope\AfterScenarioScope;

class LogsContext implements Context
{
    /** @BeforeScenario */
    public function before(BeforeScenarioScope $scope)
    {
        if (/* enable config */ === false) {
            return;
        }

        file_put_contents(
            /* filepath */,
            'START: ' . $scope->getScenario()->getTitle() . ' - ' . time() . PHP_EOL,
            FILE_APPEND
        );
    }
    
    /** @AfterScenario */
    public function after(AfterScenarioScope $scope)
    {
        if (/* enable config */ === false) {
            return;
        }

        file_put_contents(
            /* filepath */,
            'END: ' . $scope->getScenario()->getTitle() . ' - ' . time() . PHP_EOL,
            FILE_APPEND
        );
    }
}
```

## Creating the Extension

Next, we need to create the extension itself. This will serve as the entry point for our logging functionality.

```
src/
    Context/
        LogsContext.php
    ServiceContainer/
        LogsExtension.php <--- This is where we'll define our extension.
```

To ensure Behat can find and load the `LogsExtension.php` file, it's crucial to place it within the `ServiceContainer` folder. While there might be alternative ([as seen here](https://github.com/Behat/Behat/blob/5f31168f6e7c63c6c2c510bdd7772f251bea1e1c/src/Behat/Testwork/ServiceContainer/ExtensionManager.php#L176)), we'll stick to the straightforward method.

The `getConfigKey` method is used to identify our extension in the configuration, and the `configure` method is used to define the configuration tree.

```php
namespace Behat\LogsExtension\ServiceContainer;

use Behat\Testwork\ServiceContainer\Extension;

class LogsExtension implements Extension
{
    public function getConfigKey()
    {
        return 'logs_extension';
    }

    public function initialize(ExtensionManager $extensionManager)
    {
        // emtpy for our case, but useful to hook into other extensions' configuration
    }

    public function configure(ArrayNodeDefinition $builder)
    {
        $builder
            ->addDefaultsIfNotSet()
                ->children()
                    ->scalarNode('enable')->defaultFalse()->end()
                    ->scalarNode('filepath')->defaultValue('behat.log')->end()
                ->end()
            ->end();
    }

    public function load(ContainerBuilder $container, array $config)
    {
        // ... we'll load our configuration here
    }

    public function process(ContainerBuilder $container)
    {
        // emtpy for our case but needed for CompilerPassInterface
    }
}
```

## Initializing the Context

To pass configuration values to our `LogsContext`, we need to create an initializer.

```
src/
    Context/
        Initializer/
            LogsInitializer.php <--- This will handle context initialization.
        LogsContext.php
    ServiceContainer/
        LogsExtension.php
```

Here's the code for `LogsInitializer.php`:

```php
namespace Behat\LogsExtension\Context\Initializer;

use Behat\LogsExtension\Context;
use Behat\Behat\Context\Context;
use Behat\Behat\Context\Initializer\ContextInitializer;

readonly class LogsInitializer implements ContextInitializer
{
    public function __construct(
        private string $filepath,
        private bool $enable
    ) {
    }

    public function initializeContext(Context $context)
    {
        if (!$context instanceof LogsContext) {
            return;
        }

        $context->initializeConfig($this->enable, $this->filepath);
    }
}
```

We need to register the initializer definition within the Behat container through the `LogsExtension`, ensuring it gets loaded:
```php
// ...
class LogsExtension implements Extension
{
	// ...
    public function load(ContainerBuilder $container, array $config)
    {
        $definition = new Definition(LogsInitializer::class, [
            $config['filepath'],
            $config['enable'],
        ]);
        $definition->addTag(ContextExtension::INITIALIZER_TAG);
        $container->setDefinition('logs_extension.context_initializer', $definition);
    }
}
```

To complete the extension, we need to add setter methods to `LogsContext` to receive the configuration values, and use those in the hooks:
```php
// ...
class LogsContext implements Context
{
    public static bool $enable = false;
    public static string $filepath;

    public function initializeConfig(bool $enable, string $filepath)
    {
        $this->enable = $enable;
		$this->filepath = $filepath;
    }

    /** @BeforeScenario */
    public function before(BeforeScenarioScope $scope)
    {
        if ($this->enable === false) {
            return;
        }

        file_put_contents(
            $this->$filepath,
            'START: ' . $scope->getScenario()->getTitle() . ' - ' . time() . PHP_EOL,
            FILE_APPEND
        );
    }
    
    /** @AfterScenario */
    public function after(AfterScenarioScope $scope)
    {
        if ($this->enable === false) {
            return;
        }

        file_put_contents(
            $this->filepath,
            'END: ' . $scope->getScenario()->getTitle() . ' - ' . time() . PHP_EOL,
            FILE_APPEND
        );
    }
}
```

## Conclusion

Congratulations! You've just created a simple Behat extension that logs scenario timings. This extension demonstrates the three essential steps to building a Behat extension: defining an **extension**, creating an **initializer**, and configuring **contexts**.

Feel free to experiment with this extension and expand its functionality. For further learning, check out the [Behat hooks documentation](https://behat.org/en/latest/user_guide/context/hooks.html) and explore existing extensions on [GitHub](https://github.com/search?o=desc&q=behat+extension+in%3Aname%2Cdescription+language%3APHP&ref=searchresults&s=stars&type=Repositories&utf8=%E2%9C%93).

Happy testing!