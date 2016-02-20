# Rauth

[![Latest Version on Packagist][ico-version]][link-packagist]
[![Software License][ico-license]](LICENSE.md)
[![Build Status][ico-travis]][link-travis]
[![Coverage Status][ico-scrutinizer]][link-scrutinizer]
[![Quality Score][ico-code-quality]][link-code-quality]

Rauth is a simple package for parsing the `@auth-*` lines of a docblock. These are then matched with some arbitrary attributes like "groups" or "permissions" or anything else you choose. See basic usage below.

## Why?

I wanted:

- to be able to define "rules" without strapping my routes to a bunch of regex
- to be able to change routes on a whim and make the rules apply app-wide, API mode or DOM mode, without having to change anything. By putting permissions onto classes and methods, I have more control over an app's setup and its roles.

## Install

Via Composer

```bash
composer require sitepoint/rauth
```

## Basic Usage

Boostrap Rauth somewhere (preferably a bootstrap file, or wherever you configure your DI container) like so:

```php
<?php

$rauth = new Rauth();
```

*Note: you can use `setCache` or the Rauth constructor to inject a Cache object, too. Default to ArrayCache (so, inefficient and doesn't really cache anything), but can be replaced by anything that follows the Cache interface. See `src/Rauth/Cache.php`.*

Define *requirements* in `@auth-*` lines of a class or method docblock:

```php
<?php

namespace Paranoia;

/**
 * Class MyProtectedClass
 * @package Paranoia
 *
 * @auth-groups admin, reg-user
 * @auth-permissions post-write, post-read
 * @auth-mode OR
 *
 */
class MyProtectedClass
{
```

"Groups" and "Permissions" are arbitrary *attributes* a user can have - you can use "bananas" or "squirrel-hammocks" if you want. All that matters is that you separate their values with commas and that the tag starts with `@auth-`.

To check if a user has access to a given class / method:

```php
try {
    $allowed = $rauth->authorize($classInstanceOrName, $methodName, $attributes);
} catch (\SitePoint\Rauth\Exception\AuthException $e) {
    $e->getType(); // will be "ban", "and", "or", etc...
    $e->getReasons(); // an array of Reason objects with details
}
```

`$attributes` will be an array you build - this depends entirely on your implementation of user attributes. Maybe you're using something like [Gatekeeper](https://github.com/psecio/gatekeeper) and have immediate access to `groups` and/or `permissions` on a `User` entity, and maybe you have a totally custom system. What matters is that you build an array which contains the attributes like so:

```php
$attributes = [
    'groups' => ['admin']
];
```

or maybe something like this:

```php
$attributes = [
    'permissions' => ['post-write', 'post-read']
];
```

or even something like this:

```php
$attributes = [
    'groups' => ['admin', 'reg-user'],
    'permissions' => ['post-write', 'post-read']
];
```

You get the drift.

> Remember: the `@auth-*` lines are __requirements__, and they are compared against __attributes__

Rauth will then parse the `@auth` lines and save the attributes required in an array similar to that, like so:

```php
$requirements = [
    'mode' => RAUTH::OR,
    'groups' => ['admin', 'reg-user'],
    'permissions' => ['post-write', 'post-read']
];
```

`authorize` will return `true` if all is well.

If the `authorize` check fails, it will throw an `AuthException`. The `AuthException` will have a `getType` getter which will return a string value of the mode in which the failure happened - be it `ban`, `and`, `or`, `none`, or a custom mode altogether (see modes below). It will also have a `getReasons` getter which provides an array of `Reason` objects. Each object has the following public properties:

- `group`: defines which `@auth-{group}` triggered the exception, e.g. "groups", "permissions", "banana", or whatever else
- `has`: an array of the attributes provided for that group. If none were provided, empty array.
- `needs`: an array of attributes needed / prohibited, and compared against `has`.

### Available Modes

These modes can be used as values for `@auth-mode`:

#### OR

The mode `OR` will make `Rauth::authorize()` return `true` if **any** of the attributes matches **any** of the requirements.

#### AND

The mode `AND` will make `Rauth::authorize()` return `true` if **all** of the attributes match **all** of the requirements (e.g. user must have ALL the groups and ALL the permissions and ALL the bananas mentioned in the docblock).

#### NONE

The mode `NONE` will make `Rauth::authorize()` return `true` only if none of the attributes match the requirements.

## Ban

Another option you can use is the `@auth-ban` tag:

```php
/*
* ...
* @auth-ban-groups guest, blocked
* ...
*/
```

This tag will take precedence if a match is found. So in the example above - if a user is an admin, but is a member of the `blocked` group, they will be denied access. All `ban` matches MUST be zero if the user is to proceed, regardless of all other matches.

> The banhammer wields absolute authority and does not react to `@auth-mode`. Bans must be completely cleared before other permissions are even to be looked at.

## Caching

Rauth accepts in its constructor a Cache object which needs to adhere to the `src/Rauth/Cache.php` interface. It defaults to ArrayCache, which is a fake cache that doesn't really improve speed by any margin and is mainly used during development.

Note that you *can* pass in a ready-made array into the ArrayCache (constructor accepts data), if you have it. This way, you'd hydrate the cache for Rauth and it wouldn't have to manually parse every class it tries to authorize:

```php
$ac = new ArrayCache(
    [
        'SomeClass' => [
            'mode' => RAUTH::OR,
            'groups' => ['admin', 'reg-user'],
            'permissions' => ['post-write', 'post-read'],
        ],
        'SomeClass::someMethod' => [
            'mode' => RAUTH::AND,
            'groups' => ['admin'],
        ],
    ]
);

$rauth = new Rauth($ac);
```

## Best Practice

In order to avoid having to use the `authorize` call manually, it's best to tie it into a Dependency Injection container or a route dispatcher. That way, you can easily put your requirements into the docblocks of a controller, and build the attributes at bootstrapping time, and everything else will be automatic. For an example of this, see the [nofw][nofw] skeleton.

@todo This example will be added soon

## Testing

```bash
composer test
```

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md).

## Credits

- [Bruno Skvorc][link-author]

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.

[ico-version]: https://img.shields.io/packagist/v/SitePoint/Rauth.svg?style=flat-square
[ico-license]: https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square
[ico-travis]: https://travis-ci.org/sitepoint/Rauth.svg?branch=master
[ico-scrutinizer]: https://img.shields.io/scrutinizer/coverage/g/SitePoint/Rauth.svg?style=flat-square
[ico-code-quality]: https://img.shields.io/scrutinizer/g/SitePoint/Rauth.svg?style=flat-square
[ico-downloads]: https://img.shields.io/packagist/dt/SitePoint/Rauth.svg?style=flat-square

[link-packagist]: https://packagist.org/packages/sitepoint/rauth
[link-travis]: https://travis-ci.org/sitepoint/Rauth
[link-scrutinizer]: https://scrutinizer-ci.com/g/sitepoint/Rauth/code-structure
[link-code-quality]: https://scrutinizer-ci.com/g/sitepoint/Rauth
[link-author]: https://github.com/swader
[link-docs]: http://readthedocs.org