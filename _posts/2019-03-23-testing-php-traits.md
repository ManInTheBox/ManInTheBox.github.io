---
layout: post
title: "Testing PHP traits"
---

Consider the following simple demo trait:

```php
<?php

declare(strict_types=1);

namespace App\User;

trait FullNameTrait
{
    public function getFullName(string $firstName, string $lastName): string
    {
        return $firstName . ' ' . $lastName;
    }
}
```

Now, we would like to write a test case for the trait above. But this is a trait, which essentially mean that we can't really
just instantiate it like we normally do with _normal_ classes. We need another _class_ that would `use` our trait,
and **then** we would be able to test the actual behavior implemented in our trait.

So the solution is to have a class that will use our trait.

Since the only purpose of this class will be to only `use` our trait, we need something very volatile, something that
we can discard once it is used, something that we don't need to keep anymore. The solution is simple: we can use anonymous classes!


```php
<?php

declare(strict_types=1);

namespace Tests\User;

use App\User\FullNameTrait;
use PHPUnit\Framework\TestCase;

class FullNameTraitTest extends TestCase
{
    public function testFullName(): void
    {
        $subject = new class {
            use FullNameTrait;
        };

        $this->assertEquals('John Doe', $subject->getFullName('John', 'Doe'));
    }
}
```

That's it! We just tested our trait by `use`ing it in our anonymous class. Note that, here we are focused on actual trait,
and behavior provided by that trait, so we only tested methods that trait implements. The class is not important here.
