---
title: "First look at Swoole"
date: 2018-10-22T09:10:00
draft: false
---

Following on from my  [previous post on roadrunner]({{< relref "2018-09-12-Roadrunner-a-PHP-application-server.md" >}}). I've since been made aware of [Swoole](https://www.swoole.co.uk). Swoole biils itself as a "Production-Grade Async programming Framework for PHP". I confess that when I first became aware of swoole, I dismissed it as another ReactPHP/Icicle/Amp nginx fraemwork, but written as an extension.

However, if you actually [look at this diagram](https://www.swoole.co.uk/how-it-works) it shows that there is much more going on than just a async loop. The Swoole framework actually manages to make the process of many master and worker processes (as I described in Roadrunner) as seamless as a few lines of callback-like code.

Observe the simple example given in the documentation:

```php
<?php
$http = new swoole_http_server("127.0.0.1", 9501);

$http->on("start", function ($server) {
    echo "Swoole http server is started at http://127.0.0.1:9501\n";
});

$http->on("request", function ($request, $response) {
    $response->header("Content-Type", "text/plain");
    $response->end("Hello World\n");
});

$http->start();
```

This would appear to run a simple loop and run the anonymous function on HTTP request. However, reality is a bit different:

```bash
$ php test-swoole.php
Swoole http server is started at http://127.0.0.1:9501
^Z
[1]+  Stopped                 php test-swoole.php
$ ps aux | grep test-swoole | grep -v grep
scottaubrey       2500   0.0  0.0  4372096    892 s003  T     9:09am   0:00.00 php test-swoole.php
scottaubrey       2499   0.0  0.0  4363904    928 s003  T     9:09am   0:00.00 php test-swoole.php
scottaubrey       2498   0.0  0.0  4369932    620 s003  T     9:09am   0:00.00 php test-swoole.php
scottaubrey       2497   0.0  0.1  4382108  24204 s003  T     9:09am   0:00.06 php test-swoole.php
scottaubrey       2502   0.0  0.0  4363904    928 s003  T     9:09am   0:00.00 php test-swoole.php
scottaubrey       2501   0.0  0.0  4363904    900 s003  T     9:09am   0:00.00 php test-swoole.php
```

As you can see, by running the script, there are actually 6 PHP processes created. This allows the half-way house of async PHP application server that Roadrunner allows.
In fact, my early experiments with Swoole are extremely promising.

## Benchmark

I'll create another fairly useless, unscientific benchmark, using a simple, real-life web application home page (PSR-15 pipeline, router, single template, no database) under PHP built-in server, roadrunner and swoole. Using a single thread, and only 1 connection to try not disadvantage the PHP application server, we run 30 seconds of requests:

### PHP built-in server:

```bash
$ wrk -t1 -c1 -d30s http://127.0.0.1:8000
Running 30s test @ http://127.0.0.1:8000
  1 threads and 1 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     3.95ms  571.20us  14.39ms   97.05%
    Req/Sec   196.00     92.30   252.00     81.40%
  881 requests in 30.07s, 5.61MB read
  Socket errors: connect 0, read 881, write 0, timeout 0
Requests/sec:     29.30
Transfer/sec:    191.05KB
```

### Roadrunner

```bash
$ wrk -t1 -c1 -d30s http://127.0.0.1:8888
Running 30s test @ http://127.0.0.1:8888
  1 threads and 1 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   579.84us  131.64us   7.19ms   94.74%
    Req/Sec     1.71k    79.31     1.81k    85.38%
  51285 requests in 30.10s, 325.20MB read
Requests/sec:   1703.73
Transfer/sec:     10.80MB
```

### PHP built-in server:

```bash
$ wrk -t1 -c1 -d30s http://127.0.0.1:8080
Running 30s test @ http://127.0.0.1:8080
  1 threads and 1 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   166.20us   41.18us   2.72ms   97.17%
    Req/Sec     5.95k   267.90     6.21k    93.69%
  178314 requests in 30.10s, 1.11GB read
Requests/sec:   5924.05
Transfer/sec:     37.75MB
```

Remember, most of this spedd should be coming from not re-bootstrapping the application on each request. The whopping >50x faster Roadrunner becomes a whopping 4x faster still, or **200x faster that PHP application server**.

I confess, I really don't know of the qaulity of swoole's HTTP server, but the capapbility to "start" your own server, combined with the tidy performance improvement lets me think that Swoole is worth giving a try.
