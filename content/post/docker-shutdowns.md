+++
title = "Dealing With Long Docker Shutdowns"
date = "2020-06-17"
description = "Dealing with an eternity of shutdown messages, and how I fixed it"
tags = ["docker", "docker-compose", "node"]
+++


## The Problem

Have you ever had a container that seemed to take forever to shutdown? Approximately 10 seconds in fact? Well, that's the issue that I've been struggling with recently, and here's how I fixed it.

<!--more-->

This is an issue I've ran into for a long time when building Dockerfiles and using docker-compose, particularly with node (though it's not a node-unique problem). It always seemed like doing ctrl + c on the container with just `docker-compose up`, or `docker-compose down/stop` took *forever* on some containers, and I never really knew why.

## The Solution

### Solution #1

If you only need to run one command, for example `node index.js` or similar, the fix is pretty simple within a Dockerfile:

Use:
```
CMD ["node", "index.js"]
```

instead of:

```
CMD node index.js
```

Why does this work?

Well, it comes down to a few things that will be important later. First of all are "signals". Signals are simply commands sent to processes, and in this case sent to processes to stop them. When you `ctrl + c` or `docker stop`, etc., a `SIGINT` signal is sent to the process with PID (process ID) 1.

The issue with our `CMD node index.js` is that this format of `CMD` will implicitly wrap your command in `/bin/sh -c "<command>"`

Why is *that* an issue?

Essentially, the `/bin/sh` process becomes PID 1, so when SIGTERM is sent, it's only sent to `/bin/sh`, and not our node program. `/bin/sh` doesn't pass the signal along, so node doesn't get the message and it just hangs, until 10 seconds later docker gets tired of waiting and sends a `SIGKILL` signal, which goes directly to the kernel and kills all the processes. In this case, node doesn't get a chance to clean anything up, as it's treated as a rogue process, but more importantly to me... I have to wait 10 seconds.

The `CMD [...]` form works because this makes docker execute the program itself as PID 1, without wrapping it in `/bin/sh`, so it gets the signal directly, and it will shutdown nice and quickly like we want, and be given a chance to cleanup anything it wants to.

### Solution #2

So, Solution #1 helps with a lot of cases, but not if you want to 1) run multiple commands (maybe separated with `&&` or `;`) or 2) when you want to use exclusively docker-compose with no backing Dockerfile (yes, this is possible and works pretty well, and I'll be covering this more later!).

So, how do we do we use a command like `CMD yarn install && node src/index.js`, which needs to be wrapped in a to process the multiple commands, without having the 10-seconds-of-death?

It's pretty easy!

```
docker run --init ...
```

or, in docker-compose

```yml
version: '3.7'
services:
    my-container:
        image: ...
        command: /bin/sh -c "yarn install && node src/index.js"
        init: true
```
> **Note:** you need to be using docker-compose version 2.x >= 2.2, or 3.x >= 3.7

Remember how we talked about `/bin/sh` being bad about passing the `SIGINT` signal along before? That's the problem that `init` solves.

`init` uses [tini](https://github.com/krallin/tini) under the hood, which is a minimalistic "init system". The important part about `tini` for us is that it manages signals, and properly passes them along to our program. Now, when `SIGINT` is sent, it's sent to all the relevent processes and they all get a chance to happily shut down gracefully (and **quickly**).

Now you can put all the chaining commands that you want into your container and it will still work great :)
