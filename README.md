# Taiko CI

![Docker Pulls](https://img.shields.io/docker/pulls/t0mmili/taiko-ci)
![Docker Image Size](https://img.shields.io/docker/image-size/t0mmili/taiko-ci)
![GitHub License](https://img.shields.io/github/license/t0mmili/taiko-ci)

Lightweight Docker images for running [Taiko](https://taiko.dev/) browser automation tests in CI environments like GitLab or Jenkins.

## :fire: Features

- Run Taiko tests in CI out of the box
- Two variants:
  - `npm` (Node.js standard)
  - `bun` (modern, faster alternative)
- Preinstalled Chromium and dependencies (no runtime download)
- Optimized for CI performance
- Minimal configuration required

## :whale: Available Images

### Docker Hub Repository

[`t0mmili/taiko-ci`](https://hub.docker.com/r/t0mmili/taiko-ci)

### Tagging

| Tag | Description |
| --- | --- |
| `<version>-npm` | Versioned Node.js (Alpine) based image |
| `<version>-bun` | Versioned Bun (Alpine) based image |
| `latest-npm` | Latest npm-based image |
| `latest-bun` | Latest bun-based image |

> [!IMPORTANT]
> There is no generic `latest` tag.  
> Always use explicit variant tags like `latest-npm` or `latest-bun`.

## :test_tube: Example: GitLab CI

```yaml
stages:
  - tests

taiko:
  stage: tests
  image: t0mmili/taiko-ci:latest-bun
  variables:
    TAIKO_BROWSER_ARGS: '--no-sandbox,--start-maximized,--disable-dev-shm-usage'
  script:
    - taiko tests/taiko-test-1.js
```

## :gear: Configuration

Environment variables already baked into the image:

- `TAIKO_BROWSER_PATH: /usr/bin/chromium`
- `TAIKO_SKIP_CHROMIUM_DOWNLOAD: true`

Optional:

- `TAIKO_BROWSER_ARGS: '--no-sandbox,--start-maximized,--disable-dev-shm-usage'`

## :page_facing_up: License

This project is licensed under the [MIT License](https://github.com/t0mmili/taiko-ci/blob/main/LICENSE).
