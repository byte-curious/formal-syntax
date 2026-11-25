---
title: "Your First Step into Redis: A Complete Developer's Guide"
date: 2026-11-25
featured: true
author: "Anurag Mishra"
tags:  ['redis', 'caching', 'rate-limiting', 'backend']

---
Hello everyone!

This is my first post of this winter, and I thought I'd start with something I've been exploring recently: **Redis.**

I'm planning to write a series of blogs to help beginners get comfortable with Redis, BullMQ, Pub/Sub, and related concepts.

But I don't want this series to be just about reading theory or memorising definitions. I want us to understand how things work, implement what we learn, and gradually build something practical along the way.

Even though I've already worked with Redis and BullMQ in my projects, there's always more to explore than what we come across while implementing something. Sometimes, you know how to use a particular feature, but there's still more to understand about how it actually works behind the scenes. While writing these blogs, I'll also be revisiting some concepts, exploring their internals, and sharing the interesting things I discover along the way.

The idea is simple: share what I've learned so far, explore what I haven't, and hopefully make the journey useful for others too.

Let's start with Redis! 🚀

## What is Redis?

Redis stands for **Re**mote **Di**ctionary **S**erver. And no, it won't help you win at Scrabble.

It's an open-source (the code is public, free to use, and anyone can contribute), **in-memory data store**. Two ideas are packed into that phrase:

- **Data store:** a place where you can save, read, and delete data, just like a database.
- **In-memory:** the data lives in RAM (the fast, temporary working memory of your computer) instead of on disk (the slower, permanent storage where your files live).

The easiest way to picture it is a giant dictionary. You save a **value** (the data itself) under a **key** (its unique name), and you get it back by asking for that key.

```
"user:1:name"  →  "Ada"
"page:views"   →  "10482"
```

Salvatore Sanfilippo built it in 2009 because his own project was choking on slow database reads. He fixed his problem, and accidentally fixed a few million other people's too.

## Why is memory so much faster than disk?

Because RAM doesn't have to go looking for your data. It just hands it over.

| Storage | Typical access time |
|---|---|
| RAM | ~100 nanoseconds (billionths of a second) |
| SSD (the flash storage in modern laptops) | ~100 microseconds (millionths of a second) |
| Spinning hard disk | ~10 milliseconds (thousandths of a second) |

RAM is roughly **1,000x faster than an SSD**. On top of that, Redis skips fancy query planning (the database's step of working out the best way to answer a question) and works with simple data structures (organized formats for holding data, like lists or sets) right in memory. So most commands finish in well under a millisecond.

Redis also runs commands on a **single main thread** (one worker doing one job at a time). Sounds like a weakness, right? It's actually a feature. There are no locks (the "wait your turn" rules programs use when sharing data) to fight over, and every command is **atomic** (it runs completely or not at all, and nothing can sneak in halfway through). Remember this, because it saves us a lot of pain when we build the rate limiter.

## More than a key-value store

Most simple key-value stores only know about strings (plain text or numbers). Redis values can be real data structures: **strings, hashes, lists, sets, and sorted sets**. Don't worry about what those mean yet, we'll go through each one in Section 3.

The cool part is that Redis operates on them directly. "Add this player to a ranking" or "push this job onto a queue" is a single command. Your app doesn't need to fetch the data, tweak it, and save it back like it's doing homework.

## Why does Redis exist?

Databases like PostgreSQL or MySQL (popular databases that organize data into tables, like spreadsheets) store data on disk. That's safe and durable (it survives crashes and restarts), but every query (a request for data) costs time. When thousands of users ask for the same data again and again, your database becomes the **bottleneck** (the slowest part that holds everything else back) and starts sweating.

Think of a **library and your desk**. The database is the library: it has everything, but fetching a book means a walk. Redis is your desk: it only holds what you're using right now, so you grab it instantly. (Yes, my desk is messy too, but I always know where things are.)

So Redis usually doesn't replace your database. It sits **in front of it** as a fast layer.

| | Traditional database | Redis |
|---|---|---|
| Stores data in | Disk | RAM |
| Speed | Milliseconds | Sub-millisecond |
| Strength | Large, durable, queryable data | Speed and simple structures |
| Cost per GB | Low | High (your wallet will notice) |

## The cache-aside pattern

The most common use of Redis is **caching** (keeping a quick-access copy of data so you don't have to fetch it from the slow source every time). The most common way to do it is called cache-aside:

1. A request arrives.
2. The app checks Redis first.
3. **Hit** (the data is in the cache): return it immediately.
4. **Miss** (the data is not in the cache): query the database, store the result in Redis with an expiry time (a timer after which Redis deletes it automatically), then return it.

![Cache-aside pattern](/images/redis/cache-aside.svg)

Only the first request pays the price of the slow query. Everyone after that gets the answer from memory, until the cached copy expires.

## What is Redis used for?

- **Caching:** speeding up slow queries and API responses (the data your server sends back to apps)
- **Sessions:** remembering who is logged in, so users don't have to sign in on every click
- **Rate limiting:** politely telling aggressive clients to calm down (limiting how many requests one user can make)
- **Leaderboards:** real-time rankings, like a game's top players list
- **Queues:** lines of background jobs waiting to be processed, like sending emails
- **Pub/Sub** (publish/subscribe): real-time messaging where one service broadcasts and others listen

We'll build three of these in this post.

## The catch

Redis isn't free magic. RAM costs far more than disk, so you can't stuff everything into it. And by default, data in memory can vanish if the server crashes. Redis has **persistence** options (ways to save memory data to disk) for that, which we'll cover in Section 6.

There's also the old programmer joke: *"There are only two hard things in computer science: cache invalidation (knowing when a cached copy has gone stale and must be thrown out) and naming things."* Once you start caching, you'll understand why it's funny and why it hurts.

That's why I think of Redis as a **fast sidekick** to your main database, never a replacement.

## Setting up Redis

Enough theory. Let's get a real Redis running on your machine.

The easiest way is **Docker** (a tool that runs software inside isolated boxes called containers, so you don't have to install anything messy on your computer). If you don't have it yet, grab Docker Desktop from the official site. If you'd rather not use Docker at all, Redis also has a free cloud version, but we'll stick with Docker here.

Run this in your terminal:

```bash
docker run -d --name redis-demo -p 6379:6379 redis:7
```

That's one line, but it does a lot:

- `docker run`: start a new container (one running box).
- `-d`: run it in the background (detached mode), so it doesn't hog your terminal.
- `--name redis-demo`: give the container a name, so we don't have to deal with a random ID.
- `-p 6379:6379`: connect port 6379 on your computer to port 6379 inside the container. A **port** is like a numbered door that programs use to talk to each other, and 6379 is Redis's default door.
- `redis:7`: the image (a ready-made package of the software) to use, in this case Redis version 7.

The first run downloads the image, so give it a moment. Then check that it's alive:

```bash
docker ps
```

You should see `redis-demo` in the list.

## Meet redis-cli

Redis comes with `redis-cli`, a **CLI** (command line interface, meaning you type commands instead of clicking buttons) for talking to the server. Open it inside the container:

```bash
docker exec -it redis-demo redis-cli
```

You'll land on a prompt like this:

```
127.0.0.1:6379>
```

(`127.0.0.1` just means "this computer". Redis is running at home.)

Let's say hello:

```
127.0.0.1:6379> PING
PONG
```

If you see `PONG`, congratulations, you have a working Redis server. Yes, that's the whole handshake. Very dramatic, I know.

Now your first real commands:

```
127.0.0.1:6379> SET greeting "hello redis"
OK
127.0.0.1:6379> GET greeting
"hello redis"
127.0.0.1:6379> DEL greeting
(integer) 1
127.0.0.1:6379> GET greeting
(nil)
```

- `SET` saves a value under a key.
- `GET` reads it back.
- `DEL` deletes it and tells you how many keys it removed.
- `(nil)` is Redis saying "nothing here". Not an error, just an empty answer.

Type `exit` to leave the CLI. 

## Handy Docker commands

When you're done for the day:

```bash
docker stop redis-demo     # pause it
docker start redis-demo    # bring it back
docker rm -f redis-demo    # delete it completely
```

If Docker complains that port 6379 is already in use, you probably have another Redis running. Stop it, or change the left number, like `-p 6380:6379`.

Setup done. From here on, everything we do is just commands and code.

## Redis data types

Here's where Redis stops being "just a dictionary" and starts being fun. Every value in Redis has a **type** (the kind of data it holds and the commands that work on it). There are five you need to know, and honestly they cover most of what you'll ever use.

![Redis data types](/images/redis/data-types.png)

Open `redis-cli` again and follow along. Typing the commands yourself beats reading them, I promise.

### 1. Strings

The simplest type. A string holds text, a number, or even a small chunk of JSON (a common text format for sending data around). You already used `SET` and `GET`, so let's meet a more interesting command:

```
127.0.0.1:6379> SET page:views 0
OK
127.0.0.1:6379> INCR page:views
(integer) 1
127.0.0.1:6379> INCR page:views
(integer) 2
```

`INCR` (short for increment) adds 1 to a number stored in a key. Remember how I said every command is atomic (it can't be interrupted halfway)? This is where it pays off. If 1,000 users hit your site at the same moment, Redis will count all 1,000, with no lost clicks and no arguments.

**Use it for:** counters, cached values, and anything that's just "one key, one value".

### 2. Hashes

A hash is like a tiny object (a bundle of named fields) stored under one key. Think of a row in a spreadsheet.

```
127.0.0.1:6379> HSET user:1 name "Ada" age 25 city "London"
(integer) 3
127.0.0.1:6379> HGET user:1 name
"Ada"
127.0.0.1:6379> HINCRBY user:1 age 1
(integer) 26
127.0.0.1:6379> HGETALL user:1
1) "name"
2) "Ada"
3) "age"
4) "26"
5) "city"
6) "London"
```

- `HSET` sets one or more fields.
- `HGET` reads one field.
- `HINCRBY` changes a number inside one field.
- `HGETALL` returns everything (field, value, field, value...).

You could store the same user as a JSON string, but then changing one field means reading, editing, and re-saving the whole thing. With a hash, you touch only the field you care about.

**Use it for:** user profiles, settings, any "object with fields".

### 3. Lists

A list is an ordered collection of items, in the order you added them. Items can repeat.

```
127.0.0.1:6379> RPUSH tasks "write blog" "drink coffee" "deploy"
(integer) 3
127.0.0.1:6379> LRANGE tasks 0 -1
1) "write blog"
2) "drink coffee"
3) "deploy"
127.0.0.1:6379> LPOP tasks
"write blog"
127.0.0.1:6379> LLEN tasks
(integer) 2
```

- `RPUSH` adds items to the right end (the back). `LPUSH` adds to the left end (the front).
- `LRANGE tasks 0 -1` reads a slice of the list. `0` means the first item and `-1` means the last one, so this reads everything.
- `LPOP` removes and returns the item at the front.
- `LLEN` returns the length.

Push at the back, pop from the front, and you get a **queue** (a line where the first one in is the first one out, like at a chai stall). That's exactly how many background job systems work.

**Use it for:** queues, recent activity feeds, "last 10 things that happened".

### 4. Sets

A set is a collection of **unique** items, in no particular order. Add the same thing twice and Redis quietly ignores the second one.

```
127.0.0.1:6379> SADD tags "redis" "cache" "redis"
(integer) 2
127.0.0.1:6379> SMEMBERS tags
1) "cache"
2) "redis"
127.0.0.1:6379> SISMEMBER tags "redis"
(integer) 1
127.0.0.1:6379> SISMEMBER tags "sql"
(integer) 0
```

I added three items, but Redis reported `2`, because the duplicate `"redis"` was ignored. `SISMEMBER` asks "is this in the set?" and answers `1` (yes) or `0` (no). Your order from `SMEMBERS` may look different from mine, and that's normal, since sets have no order.

**Use it for:** tags, unique visitors, "who liked this post", and anywhere duplicates would be a bug.

### 5. Sorted sets

This is my favorite, and the star of our leaderboard later. A sorted set is like a set (unique items), but every item has a **score** (a number attached to it). Redis keeps the items sorted by score at all times, automatically.

```
127.0.0.1:6379> ZADD leaderboard 100 "ada" 250 "linus" 180 "grace"
(integer) 3
127.0.0.1:6379> ZREVRANGE leaderboard 0 -1 WITHSCORES
1) "linus"
2) "250"
3) "grace"
4) "180"
5) "ada"
6) "100"
127.0.0.1:6379> ZINCRBY leaderboard 100 "ada"
"200"
127.0.0.1:6379> ZREVRANK leaderboard "ada"
(integer) 1
```

- `ZADD` adds items with their scores.
- `ZREVRANGE` lists items from the **highest** score to the lowest (the `REV` means reversed). `WITHSCORES` shows the numbers too.
- `ZINCRBY` bumps a score. Ada just earned 100 points.
- `ZREVRANK` returns an item's position in the ranking, counting from 0. Ada is now `1`, which means second place.

Notice what you didn't do: you never wrote a sorting step. Redis does the sorting while inserting, so reading the top 10 stays fast even with millions of players. It's the only thing in tech that stays sorted without being asked, unlike my downloads folder.

**Use it for:** leaderboards, rankings, "top 10" lists, and anything scheduled by time or priority.

### A quick word on naming keys

You've seen names like `user:1` and `page:views`. That's a convention (an agreed habit, not a rule): use colons to group related keys, such as `user:1:name` or `cache:product:42`. This is called **namespacing** (adding a prefix so names from different features never collide). Keys are also case-sensitive, so `User:1` and `user:1` are two different keys, and yes, that will bite you at least once.

### Which one should I pick?

- One value or a counter? **String.**
- An object with several fields? **Hash.**
- Order matters, or you need a queue? **List.**
- Uniqueness matters? **Set.**
- Ranking by a number? **Sorted set.**

Redis has a few more types (streams, bitmaps, and others), but these five will carry you a long way. Next up: giving your data an expiry date.
