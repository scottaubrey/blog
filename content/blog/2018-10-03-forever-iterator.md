---
title: "Forever iterator - a good use of goto?"
date: 2018-10-03T20:00:00
draft: false
---
I had a situation recently where I was required to iterate over an array contained by an object, forever. That is to say, that I would iterate for an variable number of array items (colours, as it happens) for and unknown number of times, until I'd finished.

I thought about using the SPL iterators, specifically, an `InfiniteIterator` around an `ArrayIterator`, like so:

```php
<?php

public function getColours()
{
    return new \InfiniteIterator(new ArrayIterator($this->colours)); // where typeof($this->colours) == "array"
}

```

This would work fine, but it occurred to me that this could be a good use of goto. I remember reading a github
issue where someone submitted a PR to replace goto's in a project, and the author of the library showed that the goto's
were in fact faster, and easily understood and read. I thought this might also fit the bill. Something like this:

```php
<?php

public function getColours()
{
    start:
    yield from $this->array;
    goto start;
}
```

I actually found the second is short and quite understandable, a fairly isolated and easy to use goto.

## But, is it faster?

The SPL classes are written in C, and this situation might not match the one from that vague memory.

Quick and dirty benchmark script:

```php
<?php
function benchmark(iterable $thing, int $iterations = 1000000)
{
    $iterationsDone = 0;
    $start = microtime(true);
    foreach ($thing as $value) {
        if (++$iterationsDone == $iterations) {
            break;
        }
    }
    $end = microtime(true);

    print "time taken = ".($end-$start).PHP_EOL;
}


$array = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"];

print "benchmark InfiniteIterator".PHP_EOL;
benchmark(new \InfiniteIterator(new ArrayIterator($array)));

function foreverIterator(array $array) {
    start:
    yield from $array;
    goto start;
}


print "benchmark foreverIterator".PHP_EOL;
benchmark(foreverIterator($array));
```

The result?

``` bash
$ php test.php
benchmark InfiniteIterator
time taken = 0.14849281311035
benchmark foreverIterator
time taken = 0.038147926330566

## Fin

So there you have it. Incontrovertible evidence that the goto has a place in infinitely iterating an array in
time-sensitive applications. Was this one? No! Of course not. It was still fun though?! ;)