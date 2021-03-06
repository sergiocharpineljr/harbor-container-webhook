# harbor-container-webhook
=========

# Project Overview

harbor-container-webhook is a kubernetes mutating webhook which rewrites container images to use a Harbor proxy cache.
This is typically useful for mirroring public registries that have low rate limits, such as dockerhub, or limiting
public bandwidth usage, by mirroring images in a local Harbor registry. 

harbor-container webhook inspects pod requests in a kubernetes cluster and rewrites the container image registry of
matching images with a known proxy cache to the Harbor proxy cache location. This is achieved either thru dynamic
discovery of available proxies or static manual configuration. Both modes of running the webhook have pros & cons.

# Modes

## Static Proxy Caching
The static proxy caching mode uses configuration to tell the webhook what proxy caches exist, and what docker
registries they exist for. For example, if you had created a project in Harbor named "dockerhub-cache", and configured
the proxy cache for this project, then you could configure the static configuration as such:
```
static:
  registry_caches:
    docker.io: "harbor.example.com/dockerhub-cache"
``` 

Then the webhook would inspect every container (and init container) image, and if it matches the key, will be rewritten
to use the specified value in its place.

The static mode requires no secrets or credentials, and does not involve the webhook communicating with the Harbor API.

**Limitations:**
If you rename/delete the proxy cache project in Harbor and do not update the webhook configuration,
the webhook will misconfigure containers in your cluster, likely resulting in downtime! If this possibility concerns you, 
see the dynamic proxy caching.

## Dynamic Proxy Caching
The dynamic proxy caching mode uses the harbor API to query & cache information on all of the projects configured in
Harbor. The webhook inspects each project for any proxy-cache endpoints, and if discovered, will use the Harbor
configuration to rewrite container (and init container) images with the Harbor project's proxy cache endpoint. 

The webhook configures a TTL on the information fetched from the Harbor API and caches it. When the TTL has expired,
the webhook will continue to use the stale project information, but will issue a background request to the Harbor API
to refresh the cache. This ensures that the stale cache does not block new pods, while periodically refreshing
project metadata from Harbor. Both the cache TTL and resync interval are configurable and default to one minute.

**Limitations**:
Due to limitations in the Harbor robot account system, the dynamic proxy caching currently requires Harbor admin
credentials to query the API. These can be set via environment variables, `HARBOR_USER` and `HARBOR_PASS`.

If using the Harbor admin credentials is not appealing to you, see the static proxy caching mode.

# Example Configuration:
Example configuration for both modes of the webhook exist in docs/example-config:
* [static configuration](docs/example-config/static.yaml)
* [dynamic configuration](docs/example-config/dynamic.yaml)

# Deployment

deploy/charts contains a helm chart which can deploy the harbor-container-webhook.

# Contributing

We welcome contributions! Feel free to help make the harbor-container-webhook better.

# Code of Conduct

harbor-container-webhook is governed by the [Contributer Covenant v1.4.1](CODE_OF_CONDUCT.md)

For more information please contact opensource@indeed.com.

# License

The harbor-container-webhook is open source under the [Apache 2](LICENSE) license.
