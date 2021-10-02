# docker-build-cache-config-action [![ts](https://github.com/int128/docker-build-cache-config-action/actions/workflows/ts.yaml/badge.svg)](https://github.com/int128/docker-build-cache-config-action/actions/workflows/ts.yaml)

This is an action to generate `cache-from` and `cache-to` parameters for [docker/build-push-action](https://github.com/docker/build-push-action).


## Problem to solve

Docker BuildKit supports cache.
It imports and exports cache by the following parameters:

```yaml
cache-from: type=registry,ref=IMAGE
cache-to: type=registry,ref=IMAGE,mode=max
```

In pull request based development flow, cache efficiency would be extremely decreased.
For example,

1. Cache is created from `main` branch
1. When pull request A is opened,
    - Cache hit
    - Cache is overwritten to the branch of A
1. When pull request B is opened, cache is missed
    - Cache miss
    - Cache is overwritten to the branch of B
1. When pull request A is merged into main,
    - Cache miss
    - Cache is overwritten to `main` branch


## Solution

This action generates cache parameters for pull request based development flow.

### `pull_request` event

When a pull request is opened, this action instructs docker/build-push-action to import cache of the base branch.
It does not export cache to prevent cache pollution.
For example,

```yaml
cache-from: type=registry,ref=IMAGE:main
cache-to:
```

### `push` event of branch

When a branch is pushed, this action instructs docker/build-push-action to import and export cache of the branch.
For example,

```yaml
cache-from: type=registry,ref=IMAGE:main
cache-to: type=registry,ref=IMAGE:main,mode=max
```

### `push` event of tag

When a tag is pushed, this action instructs docker/build-push-action to import cache of the default branch.
It does not export cache to prevent cache pollution.
For example,

```yaml
cache-from: type=registry,ref=IMAGE:main
cache-to:
```

### Others

Otherwise, this action instructs docker/build-push-action to import cache of the triggered branch.
It does not export cache to prevent cache pollution.
For example,

```yaml
cache-from: type=registry,ref=IMAGE:main
cache-to:
```


## Example

Here is an example to use cache on GHCR (GitHub Container Registry).

```yaml
      - uses: docker/metadata-action@v3
        id: metadata
        with:
          images: ghcr.io/${{ github.repository }}
      - uses: int128/docker-build-cache-config-action@v1
        id: cache
        with:
          image: ghcr.io/${{ github.repository }}/cache
      - uses: docker/build-push-action@v2
        id: build
        with:
          push: true
          tags: ${{ steps.metadata.outputs.tags }}
          labels: ${{ steps.metadata.outputs.labels }}
          cache-from: ${{ steps.cache.outputs.cache-from }}
          cache-to: ${{ steps.cache.outputs.cache-to }}
```

### For monorepo

You can set a tag prefix to isolate caches.

```yaml
      - uses: docker/metadata-action@v3
        id: metadata
        with:
          images: ghcr.io/${{ github.repository }}/microservice-name
      - uses: int128/docker-build-cache-config-action@v1
        id: cache
        with:
          image: ghcr.io/${{ github.repository }}/cache
          tag-prefix: microservice-name--
      - uses: docker/build-push-action@v2
        id: build
        with:
          push: true
          tags: ${{ steps.metadata.outputs.tags }}
          labels: ${{ steps.metadata.outputs.labels }}
          cache-from: ${{ steps.cache.outputs.cache-from }}
          cache-to: ${{ steps.cache.outputs.cache-to }}
```


## Inputs

| Name | Default | Description
|------|----------|------------
| `image` | (required) | Image name to import/export cache
| `tag-prefix` | ` ` | Prefix of tag


## Outputs

| Name | Description
|------|------------
| `cache-from` | Parameter for docker/build-push-action
| `cache-to` | Parameter for docker/build-push-action
