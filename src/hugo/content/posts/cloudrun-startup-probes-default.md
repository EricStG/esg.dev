---
title: "Cloud Run startup probes default values can lead to unexpected errors"
date: 2025-05-19T12:35:00-04:00
lastmod: 2025-05-19T12:35:00-04:00
summary: Cloud Run default startup probes can lead to delays, or 429 errors when scaling
tags:
- Google Cloud
- Cloud Run
- Docker
keywords: Google Cloud, gcp, Cloud Run, Docker
---

In Cloud Run, you can configure a [Startup Probe](https://cloud.google.com/run/docs/configuring/healthchecks#startup-probes) to ensure that the service is ready. From the documentation: 

> Startup probes determine whether the container has started and is ready to accept traffic.  
When you configure a startup probe, liveness checks are disabled until the startup probe determines that the container is started, to prevent interference with the service startup.  
Startup probes are especially useful if you use liveness checks on slow starting containers, because startup probes prevents the containers from being shut down prematurely before the containers are up and running.

In practice, this can be use to ensure no traffic hits your service before it's ready to serve. You can use this to ensure your dependencies are working, or pre-populate a cache, or do whatever is it that needs to get done.

# Let's look at the default configuration for the probe.

Here's the default configuration, according to the documentation. It's a TCP probe, but it also applies if you use an HTTP probe.

```yaml
startupProbe:
  tcpSocket:
    port: CONTAINER_PORT
  timeoutSeconds: 1
  periodSeconds: 10
  failureThreshold: 3
```

Translation: I'm going to try up to 3 times, with a 1 second timeout, and 10 seconds between attempts. If your service fails the first attempt after 1 second, **it will wait for 10 seconds before retrying**.

## First issue: Waiting longer than needed

If your service takes 2 or 3 seconds to be ready? Tough luck, it won't be picked up until the second attempt. There's a good chance that your startup time will be 11 seconds instead of what you expect.

## Second issue: 429 while scaling

Cloud Run can autoscale your service based [request concurrency](https://cloud.google.com/run/docs/tips/general#optimize_concurrency). However, feature of this is that once at the limit, incoming requests will wait for the new instance to be up and running before service traffic. Again, from the documentation:

> Requests will pend for up to 3.5 times average startup time of container instances of this service, or 10 seconds, whichever is greater.

In our case, most scaling events completed in the first attempt window. But whenever we went over, it added an extra 10s, generating `429` responses from Cloud Run to our users.

# Solution: Tweak your settings

While this varies from case to case, 10 seconds for `periodSeconds` is extremely long if your users are waiting for a request to be completed. I would suggest a shorter value, maybe 2 or 3, tweaking as you need. Increasing `failureThreshold` can also help mitigate the shorter overall window.

You can use the `Container startup latency` metric provided to see how long it takes (According to Cloud Run) for your service to be available. You can use this to determine whether you may have issues with your probes.