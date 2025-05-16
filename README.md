[![License](https://poser.pugx.org/chielteuben/php-dns/license)](https://packagist.org/packages/chielteuben/php-dns)

# Fork Notes

This is a fork of remotelyliving/php-dns.
The goal is to maintain and update dependencies for compatibility and modern PHP support.
This package can be used as a drop-in replacement for the existing php-dns package with no breaking changes on the 5.x branch.

# PHP-DNS: A DNS Abstraction in PHP

### Use Cases

This library might be for you if:

- You want to be able to query DNS records locally or over HTTPS
- You want observability into your DNS lookups
- You want something easy to test / mock in your implementation
- You want to try several different sources of DNS truth
- You want to easily extend it or contribute to get more behavior you want!

### Installation

```sh
composer require chielteuben/php-dns
```

### Usage

**Basic Resolvers** can be found in [src/Resolvers](https://github.com/chielteuben/php-dns/tree/master/src/Resolvers)

These resolvers at the least implement the `Resolvers\Interfaces\DNSQuery` interface

- GoogleDNS (uses the GoogleDNS DNS over HTTPS API)
- CloudFlare (uses the CloudFlare DNS over HTTPS API)
- LocalSystem (uses the local PHP dns query function)
- Dig (Can use a specific nameserver per instance but requires the host OS to have dig installed). Based on [Spatie DNS](https://github.com/spatie/dns)

```php
$resolver = new Resolvers\GoogleDNS();

// can query via convenience methods
$records = $resolver->getARecords('google.com'); // returns a collection of DNS A Records

// can also query by any RecordType.
$moreRecords = $resolver->getRecords($hostname, DNSRecordType::TYPE_AAAA);

// can query to see if any resolvers find a record or type.
$resolver->hasRecordType($hostname, $type) // true | false
$resolver->hasRecord($record) // true | false

// This becomes very powerful when used with the Chain Resolver

```

**Chain Resolver**

The Chain Resolver can be used to read through DNS Resolvers until an answer is found.
Whichever you pass in first is the first Resolver it tries in the call sequence.
It implements the same `DNSQuery` interface as the other resolvers but with an additional feature set found in the `Chain` interface.

So something like: 

```php
$chainResolver = new Chain($cloudFlareResolver, $googleDNSResolver, $localDNSResolver);
```

That will call the GoogleDNS Resolver first, if no answer is found it will continue on to the LocalSystem Resolver.
The default call through strategy is First to Find aka `Resolvers\Interfaces\Chain::withFirstResults(): Chain`

You can randomly select which Resolver in the chain it tries first too via `Resolvers\Interfaces\Chain::randomly(): Chain`
Example:

```php
$foundRecord = $chainResolver->randomly()->getARecords('facebook.com')->pickFirst();
```

The above code calls through the resolvers randomly until it finds any non empty answer or has exhausted order the chain.

Lastly, and most expensively, there is `Resolvers\Interfaces\Chain::withAllResults(): Chain` and `Resolvers\Interfaces\Chain::withConsensusResults(): Chain`
All results will be a merge from all the different sources, useful if you want to see what all is out there.
Consensus results will be only the results in common from source to source.

[src/Resolvers/Interfaces](https://github.com/chielteuben/php-dns/tree/master/src/Resolvers/Interfaces/Chain.php)

```php
// returns the first non empty result set
$chainResolver->withFirstResults()->getARecords('facebook.com'); 

// returns the first non empty result set from a randomly selected resolver
$chainResolver->randomly()->getARecords('facebook.com'); 

// returns only common results between resolvers
$chainResolver->withConsensusResults()->getARecords('facebook.com'); 

// returns all collective responses with duplicates filtered out
$chainResolver->withAllResults()->getARecords('facebook.com'); 
```

**Cached Resolver**

If you use a PSR6 cache implementation, feel free to wrap whatever Resolver you want to use in the Cached Resolver.
It will take in the the lowest TTL of the record(s) and use that as the cache TTL.
You may override that behavior by setting a cache TTL in the constructor.

```php
$cachedResolver = new Resolvers\Cached($cache, $resolverOfChoice);
$cachedResolver->getRecords('facebook.com'); // get from cache if possible or falls back to the wrapped resolver and caches the returned records
```

If you do not wish to cache empty result answers, you may call through with this additional option:
```php
$cachedResolver->withEmptyResultCachingDisabled()->getARecords('facebook.com');
```

**Entities**

Take a look in the `src/Entities` to see what's available for you to query by and receive.

For records with extra type data, like SOA, TXT, MX, CNAME, and NS there is a data attribute on `Entities\DNSRecord` that will be set with the proper type.

**Reverse Lookup**

This is offered via a separate `ReverseDNSQuery` interface as it is not common or available for every type of DNS Resolver.
Only the `LocalSystem` Resolver implements it.

### Observability

All provided resolvers have the ability to add subscribers and listeners. They are directly compatible with `symfony/event-dispatcher`

All events can be found here: [src/Observability/Events](https://github.com/chielteuben/php-dns/tree/master/src/Observability/Events)

With a good idea of what a subscriber can do with them here: [src/Observability/Subscribers](https://github.com/chielteuben/php-dns/tree/master/src/Observability/Subscribers)

You could decide where you want to stream the events whether its to a log or somewhere else. The events are all safe to `json_encode()` without extra parsing.

If you want to see how easy it is to wire all this up, check out [the repl bootstrap](https://github.com/chielteuben/php-dns/tree/master/bootstrap/repl.php)

### Logging

All provided resolvers implement `Psr\Log\LoggerAwareInterface` and have a default `NullLogger` set at runtime. 

### Tinkering

Take a look in the [Makefile](https://github.com/chielteuben/php-dns/blob/master/Makefile) for all the things you can do!

There is a very basic REPL implementation that wires up some Resolvers for you already and pipes events to sterr and stdout

`make repl`

```sh
christians-mbp:php-dns chthomas$ make repl
Psy Shell v0.9.9 (PHP 7.2.8 — cli) by Justin Hileman
>>> ls
Variables: $cachedResolver, $chainResolver, $cloudFlareResolver, $googleDNSResolver, $IOSubscriber, $localSystemResolver, $stdErr, $stdOut
>>> $records = $chainResolver->getARecords('facebook.com')
{
    "dns.query.profiled": {
        "elapsedSeconds": 0.21915197372436523,
        "transactionName": "CloudFlare:facebook.com.:A",
        "peakMemoryUsage": 9517288
    }
}
{
    "dns.queried": {
        "resolver": "CloudFlare",
        "hostname": "facebook.com.",
        "type": "A",
        "records": [
            {
                "hostname": "facebook.com.",
                "type": "A",
                "TTL": 224,
                "class": "IN",
                "IPAddress": "31.13.71.36"
            }
        ],
        "empty": false
    }
}
=> RemotelyLiving\PHPDNS\Entities\DNSRecordCollection {#2370}
>>> $records->pickFirst()->toArray()
=> [
     "hostname" => "facebook.com.",
     "type" => "A",
     "TTL" => 224,
     "class" => "IN",
     "IPAddress" => "31.13.71.36",
   ]
>>> $records = $chainResolver->withConsensusResults()->getRecords('facebook.com', 'TXT')
{
    "dns.query.profiled": {
        "elapsedSeconds": 0.023031949996948242,
        "transactionName": "CloudFlare:facebook.com.:TXT",
        "peakMemoryUsage": 9615080
    }
}
{
    "dns.queried": {
        "resolver": "CloudFlare",
        "hostname": "facebook.com.",
        "type": "TXT",
        "records": [
            {
                "hostname": "facebook.com.",
                "type": "TXT",
                "TTL": 9136,
                "class": "IN",
                "data": "v=spf1 redirect=_spf.facebook.com"
            }
        ],
        "empty": false
    }
}
{
    "dns.query.profiled": {
        "elapsedSeconds": 0.23299598693847656,
        "transactionName": "GoogleDNS:facebook.com.:TXT",
        "peakMemoryUsage": 9615080
    }
}
{
    "dns.queried": {
        "resolver": "GoogleDNS",
        "hostname": "facebook.com.",
        "type": "TXT",
        "records": [
            {
                "hostname": "facebook.com.",
                "type": "TXT",
                "TTL": 21121,
                "class": "IN",
                "data": "v=spf1 redirect=_spf.facebook.com"
            }
        ],
        "empty": false
    }
}
{
    "dns.query.profiled": {
        "elapsedSeconds": 0.0018258094787597656,
        "transactionName": "LocalSystem:facebook.com.:TXT",
        "peakMemoryUsage": 9615080
    }
}
{
    "dns.queried": {
        "resolver": "LocalSystem",
        "hostname": "facebook.com.",
        "type": "TXT",
        "records": [
            {
                "hostname": "facebook.com.",
                "type": "TXT",
                "TTL": 25982,
                "class": "IN",
                "data": "v=spf1 redirect=_spf.facebook.com"
            }
        ],
        "empty": false
    }
}
=> RemotelyLiving\PHPDNS\Entities\DNSRecordCollection {#2413}
>>> $records->pickFirst()->getData()->getValue()
=> "v=spf1 redirect=_spf.facebook.com"
>>> 
```