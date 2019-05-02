---
layout: post
title: "Switchable Logger"
---

Have you ever wanted to have the ability to dynamically, at run-time, control the logging in your application?

Let's suppose you're building some network-based application, like simple client-server chat application. Normally,
your application will perform all sorts of tasks typical for chat applications like connect/disconnect, log in/log out,
send message/receive message, etc. You probably want to record all or most of these significant events, that is to _log_ them, just in
case if you need them in the future, for whatever reason. However, soon you realize that your application is generating really a 
lots of logs, and you're very quickly reaching your monthly plan at your logging service. Then you decide either to
turn the logging off entirely, or to use some other strategy to log only some smaller percentage of all available logs
(like [`Monolog\SamplingHandler`](https://github.com/Seldaek/monolog/blob/4a33226f25009758cb237b4383589cef023b9494/src/Monolog/Handler/SamplingHandler.php)). 

What if you really don't actually need any logs, most of the time? It would be super cool to keep the logging turned off all the time,
and turn it on, only when you need it. Once you're done, you turn it off again, until the next time you need the same thing.

_Switchable Logger_ is a simple solution designed to solve the problem explained above. Its sole purpose is to log events
only when enabled.

Let's see how `SwitchableLogger` could be implemented. _Credits for class name `SwitchableLogger` go to my colleague Ben._

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Logging;

use Psr\Log\AbstractLogger;
use Psr\Log\LoggerInterface;

final class SwitchableLogger extends AbstractLogger
{
    /**
     * @var LoggerInterface
     */
    private $logger;

    /**
     * @var LoggingStateInterface
     */
    private $loggingState;

    public function __construct(LoggerInterface $logger, LoggingStateInterface $loggingState)
    {
        $this->logger = $logger;
        $this->loggingState = $loggingState;
    }

    public function log($level, $message, array $context = [])
    {
        if (!$this->loggingState->isEnabled()) {
            return;
        }

        $this->logger->log($level, $message, $context);
    }
}
```

`SwitchableLogger` simply [_decorates_](https://en.wikipedia.org/wiki/Decorator_pattern) any [PSR-3](https://www.php-fig.org/psr/psr-3/)-compliance
logger implementation and performs the real logging if its _"logging state"_ is enabled. The sole purpose of _"logging state"_
passed to this logger is to tell us whether or not logging is enabled.

You can notice that _"logging state"_ is represented as an abstration named `LoggingStateInterface`:

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Logging;

interface LoggingStateInterface
{
    public function isEnabled(): bool;
}
```

This is how we encapsulate algorithms for determining whether or not logging should be enabled. We also decouple them from
the client code `SwitchableLogger`. This is, in essence, simple [Strategy pattern](https://en.wikipedia.org/wiki/Strategy_pattern) implementation.
All we have to do is to implement `LoggingStateInterface` and decide when we want to enable logging. Let's do some quickly!

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Logging;

use Redis;

class RedisControlledLoggingState implements LoggingStateInterface
{
    private const REDIS_KEY = 'logging.enabled';

    /**
     * @var Redis
     */
    private $redis;

    public function __construct(Redis $redis)
    {
        $this->redis = $redis;
    }

    public function isEnabled(): bool
    {
        return (bool) $this->redis->get(self::REDIS_KEY);
    }
}
```

`RedisControlledLoggingState` is a logging state that can be controlled with Redis. Simply set `logging.enabled` key in Redis
and logging will be enabled. Delete the key and logging is disabled.


```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Logging;

class RandomSelectionLoggingState implements LoggingStateInterface
{
    public function isEnabled(): bool
    {
        return (bool) random_int(0, 1);
    }
}
```

Ok, this is just a silly one for demonstration purposes. `RandomSelectionLoggingState` randomly turns the logging on or off.

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Logging;

class NightShiftLoggingState implements LoggingStateInterface
{
    private const START = 21;
    private const END = 7;

    public function isEnabled(): bool
    {
        return date('H') >= self::START || date('H') <= self::END;
    }
}
```

Yet another dumb example of possible logging state implementations. `NightShiftLoggingState` ensures logging is turned on
while everyone's sleeping.

Now, let's see very quickly how to use all of these in the real world example. We'll use [`Monolog`](https://github.com/seldaek/monolog)
as PSR-3 logger implementation, but it can be any other that conforms to PSR-3.

```php
<?php

$logger = new SwitchableLogger(new Monolog\Logger(), new NightShiftLoggingState());
$logger->warning('No one is here during the night! Time for logging.');

$logger = new SwitchableLogger(new Monolog\Logger(), new RandomSelectionLoggingState());
$logger->info('This might or might not be logged...');

$logger = new SwitchableLogger(new Monolog\Logger(), new RedisControlledLoggingState());
// redis-cli> SET logger.enabled 1
$logger->debug(sprintf('Huge JSON %s received over the network.', $json));
// redis-cli> DEL logger.enabled
$logger->debug(sprintf('Again, enormous JSON %s received but the logging is turned off.', $json));
```

`SwitchableLogger` is a very simple yet powerful implementation that solves one concrete problem. The code presented
in this blog post is also available on [Github](https://github.com/ManInTheBox/switchable-logger). Feel free to download
and use it in your projects if you think it could be useful for you.
