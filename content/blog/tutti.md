+++
title = "Local Development with Microservices"
date = 2025-12-03
description = "Containerless process orchestrating"
draft = false
tags = ["rust"]
+++

Recently i faced a problem with local development of microservices. When you have a lot of them, managing everything on your machine becomes painful. If you need to run all services at once, it quickly becomes a puzzle: remember who depends on whom.

Imagine you are building a book store and have several microservices. Your dependency graph might look like this:

{% mermaid() %}
graph TD;
  Catalog --> Auth;
  Reviews --> Catalog;
  Reviews --> Auth;
  Pricing --> Catalog;
  Cart --> Pricing;
  Cart --> Catalog;
  Cart --> Auth;
  Orders --> Catalog;
  Orders --> Auth;
{% end %}

So you have a bunch of microservices that depend on each other. Cart service could synchronously go to the Pricing service, and asynchronously receive events from the Catalog service.

Now consider this situation: you add a new feature to the Cart microservice and want to test it locally. To do that properly, you must run **all** its dependencies. Yes, you can use docker-compose to start everything at once. It's easy you should describe all services in a single `docker-compose.yml` file with `depends_on` attributes, just run `docker-compose up` and... wait until an image is built. When you modify a microservice, you need to rebuild its image. This can be slow. On my Macbook it becomes even more annoying because I cannot simply mount a volume with a locally compiled binary for each microservice. The binary is compiled for macOS, while Docker runs on Linux. So i must compile it inside Docker.

So I came up with a solution. I wrote an application called [Tutti](https://github.com/ya7on/tutti) that solves this problem. Tutti is a containerless process orchestration tool that lets you start and stop microservices locally without building containers. It also lets you mount volumes with locally compiled binaries for each microservice, so you can test your changes without rebuilding Docker images every time.

Back to our example with book store. You should just describe all services in a single `tutti.toml` configuration file:

```toml
[service.catalog]
cmd = ["cargo", "run", "--bin", "catalog"]
deps = ["auth"]

[service.auth]
cmd = ["cargo", "run", "--bin", "auth"]

[service.reviews]
cmd = ["cargo", "run", "--bin", "reviews"]
deps = ["catalog", "auth"]

[service.pricing]
cmd = ["cargo", "run", "--bin", "pricing"]
deps = ["catalog"]

[service.cart]
cmd = ["cargo", "run", "--bin", "cart"]
deps = ["pricing", "catalog", "auth"]

[service.orders]
cmd = ["cargo", "run", "--bin", "orders"]
deps = ["catalog", "auth"]
```

And that's it! You can now start and stop your microservices locally without building containers or mounting volumes with locally compiled binaries. Just by running:

```
tutti-cli run catalog
```
