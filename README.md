# Handling optional dependencies with composer

Optional dependencies is something you normally should avoid in any case of
a library where it is possible. In this case we first look at alternatives
and then how we still could work with optional dependencies.

## Why are optional dependencies bad?

The main reason why optional dependencies are bad is that you can not be sure
that the dependency which is installed by the project matches the version which
your library is compatible with.

It is common that if you work with `optional dependencies` already that you did define
the requirement of your `optional dependency` in the `require-dev` section. But as example
if you have:

```json
{
    "require-dev": {
        "elasticsearch/elasticsearch": "^3.0"
    }
}
```

A project installing your package will not fail when it has in its composer.json:

```json
{
    "require": {
        "elasticsearch/elasticsearch": "^4.0"
    }
}
```

Another good example about optional dependencies is for example the [`doctrine/orm`](https://github.com/doctrine/orm/).
When you install the `doctrine/orm` package the installation will not fail when
you don't have a maybe required `pdo`, `pgsql` or `mysql` extension not installed.
That is about the nature how doctrine/orm is build as a whole package with different
drivers in it, and so they can not define the `require` section as different drivers
require different things. So it does not make sense in `doctrine/orm` to require all
used extensions as only one driver is used mostly inside a project.

The alternative here to `optional` dependencies would be to split a package
into multiple packages via as example
[Adapter Pattern](https://en.wikipedia.org/wiki/Adapter_pattern).
A good example for this is the [`league/flysystem`](http://github.com/league/flysystem) package. That package
provides different adapters which have their own `requirements` and so exactly
tells you with which already installed packages it is compatible or not. As an example
it provides a `league/flysystem-aws-s3-v3` adapter which requires a specific version of
`aws/aws-sdk-php` and as that is an own package it make sure that you have the correct version of
`aws/aws-sdk-php` installed.

## What does composer say about optional dependencies?

In the official documentation you will not find any statement about `optional dependencies`.
But if you search in the issues of composer you will find a [`RFC` issue for `optional dependencies`](https://github.com/composer/composer/issues/8184).

I want to quote here the answer from [Seldaek](https://github.com/composer/composer/issues/8184#issuecomment-501265594)

> [Seldaek](https://github.com/Seldaek) commented on [Jun 12, 2019](https://github.com/composer/composer/issues/8184#issuecomment-501265594)  
>   
> Monolog is a very good example of doing it wrong really..  
> ElasticSearchHandler should be published as a standalone package with a requirement on both monolog and elasticsearch/elasticsearch, so the version can be enforced. That is why I don't accept any new handlers anymore with external requirements.
>   
> Adding optional dependencies are not worth the trouble IMO in composer.

That was very eye-opening for me why in general optional dependencies are wrong and that
kind of things should be split into own packages where possible.

## The composer `provide` feature

Composer supports something which is called `virtual packages`.
A good example for this is the `php-http/client-implementation` package:

```json
{
    "require": {
        "php-http/client-implementation": "^1.0"
    }
}
```

This package is a virtual package and will require a package which provides the
`php-http/client-implementation` implementation. And the end user of your package then
is required to install a package which requires this implementation. A package doing
this tells packagist/composer this via:

```json
{
    "provide": {
        "php-http/client-implementation": "1.0"
    }
}
```

To recommend a specific implementation you can use the `suggest` section:

```json
{
    "suggest": {
        "symfony/http-client": "For using Symfony as HTTP client"
    }
}
```

In case of the `php-http/client-implementation` the `php-http/discovery` package
exist which is since `1.15` also a composer plugin which automatically installs
the best matching library and even don't need to handle yourself. Thanks goes here
to [nicolas-greaks](https://github.com/php-http/discovery/pull/208) providing this
great feature.

For optional packages you could also use the `provide` section to create your own
virtual packages and provide packages for it.

If you want avoid accidentally created optional dependencies you should have a look at
[maglnet/ComposerRequireChecker](https://github.com/maglnet/ComposerRequireChecker) plugin.

## How to handle optional dependencies with composer

There are maybe situation where you just don't want to split your package up. There
could be different reasons for this that one package with optional packages provides
a better developer experience as that the enduser of your library need to install 
several packages.

In my situation while working on the [`schranz-search/schranz-search`](https://github.com/schranz-search/schranz-search) packages. I did
go the `flysystem` way and using the
[Adapter Pattern](https://en.wikipedia.org/wiki/Adapter_pattern) to split the support
of different search engines into different packages. So good so fine no optional
dependencies needed as every package itself requires the needed dependencies.

But now I did come to the situation about integrating the different packages into
different Frameworks. In my example I wanted to create a `Symfony Bundle` which
integrates the library into Symfony Ecoystem. If I would be strict about optional
dependencies here I would have needed to create for every adapter an own bundle.
Also I did not wanted to require all adapters in the `require` section of my bundle
as that would install a lot of dependencies and maybe even dependencies of dependencies
between adapters are not even compatible with each other.

As creating multiple bundles per adapters and I want to go an easy replaceable `DSN`
way of configuring the bundle and make so changing between adapters easy. I needed
some kind of optional dependencies.

So to test my bundle against all adapters I did add them as `require-dev` to my
`composer.json`. But as mention above this will not avoid that somebody is in future
installing a wrong version of an adapter which is maybe not compatible with the
installed bundle version. So to fix that issue we also add the `conflict` section to
our `composer.json`:

```json
{
    "require-dev": {
        "schranz-search/seal-algolia-adapter": "^0.3",
        "schranz-search/seal-elasticsearch-adapter": "^0.3",
        "schranz-search/seal-meilisearch-adapter": "^0.3",
        "schranz-search/seal-memory-adapter": "^03",
        "schranz-search/seal-multi-adapter": "^0.3",
        "schranz-search/seal-opensearch-adapter": "^0.3",
        "schranz-search/seal-read-write-adapter": "^0.3",
        "schranz-search/seal-redisearch-adapter": "^0.3",
        "schranz-search/seal-solr-adapter": "^0.3",
        "schranz-search/seal-typesense-adapter": "^0.3"
    },
    "conflict": {
        "schranz-search/seal-algolia-adapter": "<0.3 || >=0.4",
        "schranz-search/seal-elasticsearch-adapter": "<0.3 || >=0.4",
        "schranz-search/seal-meilisearch-adapter": "<0.3 || >=0.4",
        "schranz-search/seal-memory-adapter": "<0.3 || >=0.4",
        "schranz-search/seal-multi-adapter": "<0.3 || >=0.4",
        "schranz-search/seal-opensearch-adapter": "<0.3 || >=0.4",
        "schranz-search/seal-read-write-adapter": "<0.3 || >=0.4",
        "schranz-search/seal-redisearch-adapter": "<0.3 || >=0.4",
        "schranz-search/seal-solr-adapter": "<0.3 || >=0.4",
        "schranz-search/seal-typesense-adapter": "<0.3 || >=0.4"
    }
}
```

This way even `composer` is not supporting directly optional dependencies, we can
over the `conflict` section of `composer.json` make sure that only compatible versions
of our optional requirements are installed. A good website to find the correct constraint for your `conflict` part is [https://semver.madewithlove.com/](https://semver.madewithlove.com/).

Over `suggest` or with  require an `implementation` like in the above section [here](#the-composer-provide-feature),
composer would also tell the enduser which adapters exist or are suggested by us to be used with our integration bundle.

I hope I could help to understand how to avoid optional dependencies and how to handle them with composer.

If you have any feedback or questions add it to the Twitter Thread [here](https://twitter.com/alex_s_/status/1640429500763611153) or write an issue on [this](https://github.com/alexander-schranz/composer-optional-dependencies/issues/new) Repository.
If you liked the article give it a star here or even retweet it :).

Interested in other articles? Go to the [`alexander-schranz-article` topic](https://github.com/topics/alexander-schranz-article).
