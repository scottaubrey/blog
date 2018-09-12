---
title: "Roadrunner: a PHP application server"
date: 2018-09-12T13:10:00
draft: false
---

During the beta phase for PHP 5.4, I was really excited about the built in PHP application server. Finally, I no longer had to keep hosts files updated, or web servers, or FastCGI servers. At a time when other PHP professionals were moving to Vagrant for re-producable dev environments and other programming languages had defined application interfaces such as Rack and WSGI, I saw the built in PHP server as a way to make simple PHP applications easy to get up and running without the configuration overhead of apache, nginx or PHP-FPM. This was a benefit adopted by some frameworks (Symfony's `bin/console server:run`) which utilise the same for their CLI serve commands, and I adopted it for all the projects I was involved in.

Docker was mere months away from it's first release, let alone a production ready release, but as that app-container metality took off (not the replacement for Vagrant that more often gets attention), you can see the value in being able to create a container for your app, with a single entrypoint and simple configuration.

But, it was, and still is only really suitable for development. It's single threaded, it was quite buggy (particularly early versions) and it lacks features required of a web server. I dreamed of using a set of simple PHP application server that could be ran the same in dev and production, containers or otherwise.

## Roadrunner: a Golang based PHP Application Server

I subscribe to dev.to's feature articles list, and found myself browsing random around golang related projects that might connect a running golang application to PHP (beyond the obvious HTTP-based APIs). That's when I came across [Roadrunner](https://github.com/spiral/roadrunner) on github. Roadrunner bills itself as a PSR7 application server, but in practice perfectly fulfils my desire for a simple PHP application server.

Roadrunner works by creating a HTTP server with golang's excellent net/http package, and using [Goridge](https://github.com/spiral/goridge) as a bridge between golang and PHP to pass PSR7 Request and Responses between PHP and golang. The PHP application is then a long-running, already bootstrapped PSR7-capable application that received the already parsed PSR7 request, dispatches it, and collects the response to give back into golang. The effect is really quite simple, and in a well written modern application should not hold too many caveats.

Here's a cut down version of my usual `index.php` file for running on PHP Development server, utilising Zend Diactoros for PSR7 Request parsing, and Zend HTTPHander for response emitting:

```php
<?php
//boostrap application
$psr7Application = require 'bootstrap.php';

//create a request from PHP's environment
$request = Zend\Diactoros\ServerRequestFactory::fromGlobals();

//dispatch application
$response = $psr7Application->handle($request);

//emit response to PHP environment
(new Zend\HttpHandlerRunner\Emitter\SapiEmitter)->emit($response);
```

and the same but with Roadrunner's request parser and response emitter:

```php
<?php
//boostrap application
$psr7Application = require 'bootstrap.php';

//create Roadrunner worker
$worker = new RoadRunner\Worker(new Goridge\StreamRelay(STDIN, STDOUT));
$psr7Worker = new RoadRunner\PSR7Client($worker);

//in a loop wait for a request to come in from Roadrunner's server
while ($request = $psr7Worker->acceptRequest()) {
    //dispatch application
    $response = $psr7Application->handle($request);

    //emit response to Roadrunner
    $psr7Worker->respond($response);
}
```

The golang part of the equation does the heavy lifting in starting, managing and stopping your application, and controls how many worker processes to run. It can also acts serve static files too. The config is a single YAML file (`.rr.yaml`), for example:

```yaml
http:
  enable:    true
  address:   127.0.0.1:8888
  maxRequest: 200
  uploads:
    forbid: [".php", ".exe", ".bat"]
  workers:
    command:  "php rr-worker.php pipes"
    relay:    "pipes"
    pool:
      numWorkers: 4
      maxJobs:  0
      allocateTimeout: 60
      destroyTimeout:  60

static:
  enable:  true
  dir:   "public"
  forbid: [".php", ".htaccess"]
```

Then run `rr` golang binary and... it works. I was really surprised that this actually worked first time once I had written the above files from their example documentation.

## What's the point?

Well, in our containerised world, I think the requirements for a PHP container become PHP CLI, your source code, and the Roadrunner binary. Wow! No nginx, apache etc. So much smaller and easier to understand as a single container.

It's also wickedly fast. As an illustration only I've run a stupidly obtuse single-connection benchmark for a simple Zend Stratigility application, with Twig rendered response. Firstly using the PHP dev server:

```bash
$ wrk -t1 -c1 -d30s http://127.0.0.1:8080
Running 30s test @ http://127.0.0.1:8080
1 threads and 1 connections
Thread Stats   Avg      Stdev     Max   +/- Stdev
Latency     2.76ms  418.23us   7.15ms   92.55%
Req/Sec   228.61    146.49   373.00     69.41%
2229 requests in 30.09s, 13.19MB read
Socket errors: connect 0, read 2229, write 0, timeout 0
Requests/sec:     74.08
Transfer/sec:    448.73KB
```

The same using Roadrunner:

```bash
$ wrk -t1 -c1 -d30s http://127.0.0.1:8888
Running 30s test @ http://127.0.0.1:8888
  1 threads and 1 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   286.67us  768.79us  33.85ms   99.78%
    Req/Sec     3.81k   244.98     4.00k    90.03%
  114162 requests in 30.10s, 672.40MB read
Requests/sec:   3792.70
Transfer/sec:     22.34MB
```

50 times the number of requests!

The bulk of this saving is because you only start PHP and your application once, then reuse the application for multiple requests, where as usually PHP tears down and rebuilds the entire PHP application for each request. If we benchmarked with multiple threads and connections, and tweaked the number of workers in Roadrunner, this value would be even higher (just increasing the connections/threads got to >6500 requests/sec on my laptop)

And, although I haven't used it yet, there's oppourtunity to use Roadrunner's middleware functionality to utilise golang's speed to preprocess other parts of your application. Have authentication done before you get to PHP, or sessions, or authorisation. You can also use the goridge RPC to offload other processing within a request to golang, which may or may not be useful, like a really fast task queue into go.

## What's the catch?

There are some disadvantages:

### 1. Not every application can run this way

If you application uses PHP's GLOBALS (such as \$\_GET), this isn't going to work. PHP's built in session module doesn't work. You application has to be 100% PSR7 request and response driven, and that's not the norm. I would guess most framework-driven applications strive for this, but it's easy to be complacent when it doesn't matter, and just grab a \$\_GET variable here and there. I've also seen many PSR-7 application middleware that uses the session module anyway. This won't work, the PHP environment that would usually handle this isn't running. It has be to be driven by PSR7 Request/Response objects.

You also gonna need to watch your resources. In particular, anything that you rely on the PHP environment being destroyed to close. Global-scoped variables, caches, connections. None of these will go through the usual PHP teardown, and so you'll need to be careful about using global scope, memory usage, and resources. Be particularly aware of tools like Doctrine, that can keep huge caches of data in memory (in the identity map for example) needing to be cleared after a request completes. Roadrunner has *some* compensatory tools, such as setting the worker config `maxJobs` to have them restarted after X number of requests, but you'll still need to be careful. .

### 2. The usual edit/save/refresh cycle doesn't work

Because you're starting and running your application once to be reused for multiple requests, if you edit a file that has already been loaded, PHP is going to have the old one in memory and bootstrapped. You can't just edit the file, save and refresh.

Other language's application servers also have this issue, and often will have tooling to watch for file events and hot-reload either part or entire applications for development. Roadrunner has a reload command, but you're going to have to provide the watcher yourself. I used the excellent `modd` with a config:

    **/*.php {
        prep: rr http:reset
    }

I accept this does negate my previous statement about this being simpler than setting up a full FPM/nginx/apache, and is actually more moving parts during dev than PHP's built in development server. I'd like to see this functionality incorporated into the core of Roadrunner (which may be my first golang contribution in the future).

## What about ReactPHP/Icicle/Amp/PHPPM/Other PHP-based application server?

While I'd appreciate a PHP-based application server, it's not ideal for a number of reasons:

1. PHP just doesn't have the same engineering into making it run well in long-lasting applications. To make your core HTTP server run in PHP is asking for trouble IMO.
2. Each of these "servers" tend to be more of the node.js/event-loop style of application server, which require you to restructure your application to embed these concepts and truly benefit. There's less need for that with Roadrunner (though you cannot plug in any PHP code, as above).
3. PHPPM seems to be the closest equivalent of Roadrunner, but the connection seems overly engineered, seemingly generating connection code and thus only really supporting framework defaults.
4. The golang net/http package provides a rock solid HTTP server and Roadrunner brings that production quality to PHP. These other servers just don't have that kind of backing and hardening.

## Conclusion

While not for everyone and even application, and there are certainly more challenges, I'm really quire excited by this project, and wonder why isn't not gaining much more attention. The performance benefits are astounding, and without much altering of a modern PHP application, whilst potentially allowing PHP to enter the brave world of application containers in a simpler way.

