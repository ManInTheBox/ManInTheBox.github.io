---
layout: post
title: "How to inline a service in Symfony"
---

With Symfony's [DependencyInjection] component it is possible to inline service definitions.
_Inlining_ a service means to define the service right away, for one-off usage, usually as the argument of another service.

A classic example would be the service which requires an argument, which is in turn also a service. But in order to
inject that other service into the main service, you need to first define it as a service - this is where inlining a service is useful!

Here's a quick code example that explains what we want to achieve here, using the well-known `Mailer` and `NewsletterManager` examples
from the official Symfony documentation.

Imagine we have a `Mailer` class that looks like this:

```php
<?php

namespace App\Email;

class Mailer
{
    private $transport;

    public function __construct(string $transport = 'sendmail')
    {
        $this->transport = $transport;
    }
}
```

And we also have a `NewsletterManager` service which depends on the `Mailer` instance:

```php
<?php

namespace App\Email;

class NewsletterManager
{
    private $mailer;

    public function __construct(Mailer $mailer)
    {
        $this->mailer = $mailer;
    }
}
```

Let's use YAML to define `NewsletterManager` as a service and its dependency `Mailer` as inlined service:

```yaml
services:
    App\Email\NewsletterManager:
        arguments:
            - !service { class: App\Email\Mailer }
```

That's it! You just inlined `Mailer` service with the help of custom YAML tag `service`. As you might be guessing
`!service` accepts all options you would normally use with any other service definition. For example, if you want to pass
argument `$transport` to the inlined `Mailer` service:

```yaml
services:
    App\Email\NewsletterManager:
        arguments:
            - !service { class: App\Email\Mailer, arguments: [sendmail] }
```

Or if you have a `MailerFactory` which instantiates `Mailer` objects:

```php
<?php

namespace App\Email;

class MailerFactory
{
    public static function create(string $transport = 'sendmail'): Mailer
    {
        return new Mailer($transport);
    }
}
```

```yaml
services:
    App\Email\NewsletterManager:
        arguments:
            - !service { class: App\Email\Mailer, factory: [App\Email\MailerFactory, create] }
```

If you prefer XML, here's the syntax:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<container xmlns="http://symfony.com/schema/dic/services"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/services
        http://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>
        <service id="App\Email\NewsletterManager">
            <argument type="service">
                <service class="App\Email\Mailer"/>
            </argument>
        </service>
    </services>
</container>
```

Or if you need a factory in order to instantiate `Mailer`:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<container xmlns="http://symfony.com/schema/dic/services"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/services
        http://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>
        <service id="App\Email\NewsletterManager">
            <argument type="service">
                <service class="App\Email\Mailer">
                    <factory class="App\Email\MailerFactory" method="create"/>
                </service>
            </argument>
        </service>
    </services>
</container>
```

That's all! Of course, you could always use private services to achieve the same goal and they would be removed
during the container optimization and cleanup phase by Symfony. Inlined services are just an alternative way and I hope
that you learned now how to easily employ that useful feature.

[DependencyInjection]: https://symfony.com/doc/current/components/dependency_injection.html
