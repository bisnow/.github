# Bisnow Shared GitHub Configuration

This repository contains shared configuration, reusable workflows, and composite actions for all repositories within the [Bisnow organization](https://github.com/bisnow).

> Run `bisnow repo:init` from [bisnow/local](https://github.com/bisnow/local) to ensure your repository has all the required Composer scripts and configuration files for these workflows.

---
- [Reusable Workflows](#reusable-workflows)
    - [Laravel Tests](#laravel-tests)
        - [Basic Usage](#basic-usage)
        - [With Multiple Databases](#with-multiple-databases)
        - [With Frontend Build](#with-frontend-build)
        - [With OpenSearch](#with-opensearch)
        - [With PostgreSQL](#with-postgresql)
        - [With Docker (Specific Versions)](#with-docker-specific-versions)
        - [With Flux Pro](#with-flux-pro)
        - [Full Example](#full-example)
        - [Workflow Inputs Reference](#workflow-inputs-reference)
        - [Schema Imports](#schema-imports)
        - [Environment Variables](#environment-variables)
        - [Runner Service vs Docker](#runner-service-vs-docker)
        - [Pipeline](#pipeline)
    - [Laravel Code Styling](#laravel-code-styling)
        - [Basic Usage](#basic-usage-1)
        - [With Flux Pro](#with-flux-pro-1)
        - [With a Specific PHP Version](#with-a-specific-php-version)
        - [How It Works](#how-it-works)
        - [Workflow Inputs Reference](#workflow-inputs-reference-1)
        - [Using Both Workflows Together](#using-both-workflows-together)
- [Composite Actions](#composite-actions)
    - [setup-mysql](#setup-mysql)
    - [setup-opensearch](#setup-opensearch)
    - [setup-pgsql](#setup-pgsql)
    - [setup-php-composer](#setup-php-composer)
    - [setup-node-npm](#setup-node-npm)
    - [setup-redis](#setup-redis)
- [Shared Configuration](#shared-configuration)
    - [Pull Request Templates](#pull-request-templates)
    - [EditorConfig](#editorconfig)
- [Versioning](#versioning)
---

## Reusable Workflows

### Laravel Tests

The `tests-laravel.yml` workflow runs PHPStan static analysis and PHPUnit/Pest tests for Laravel applications. It handles service setup, dependency caching, and environment preparation automatically.

#### Basic Usage

At minimum, you only need to provide the `COMPOSER_OAUTH_GITHUB_ACTIONS` secret:

```yaml
# .github/workflows/tests.yml
name: Tests
on: [push]

jobs:
  tests:
    uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

This gives you PHP 8.4, MySQL (runner service), and Redis out of the box.

#### With Multiple Databases

Many applications require additional databases for multi-tenancy or service isolation:

```yaml
jobs:
  tests:
    uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2
    with:
      mysql_db_name: bisreach
      mysql_db_names: '["contacts", "forage", "tidy", "salutary"]'
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

> **Note:** The `mysql_db_names` input accepts a JSON array. Invalid JSON will fail fast with a clear error message.

#### With Frontend Build

For applications that require Node.js asset compilation:

```yaml
jobs:
  tests:
    uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2
    with:
      enable_npm: true
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

Node reads the version from `.nvmrc` by default. Supported version files:

- `.nvmrc`
- `.node-version`
- `.tool-versions`
- `package.json` (via the `engines.node` field)

To use a different file:

```yaml
with:
  enable_npm: true
  node_version_file: '.node-version'
```

#### With OpenSearch

For applications that need a search engine:

```yaml
jobs:
  tests:
    uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2
    with:
      enable_opensearch: true
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

OpenSearch runs as a single-node Docker container with security disabled, suitable for testing.

#### With PostgreSQL

For applications using PostgreSQL alongside or instead of MySQL:

```yaml
jobs:
  tests:
    uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2
    with:
      enable_pgsql: true
      pgsql_db_name: warehouse
      pgsql_db_names: '["analytics"]'
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

#### With Docker (Specific Versions)

By default, MySQL and PostgreSQL use the pre-installed runner services for faster startup. When you need a specific version, enable Docker mode:

```yaml
jobs:
  tests:
    uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2
    with:
      mysql_use_docker: true
      mysql_version: '8.0.35'
      enable_pgsql: true
      pgsql_use_docker: true
      pgsql_version: '16'
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

The `pgsql_image` and `pgsql_version` values correspond to Docker image tags. When using a custom image like pgvector, the tag format may differ:

```yaml
# Standard postgres image: postgres:16
pgsql_image: postgres
pgsql_version: '16'

# pgvector image: pgvector/pgvector:pg16
pgsql_image: pgvector/pgvector
pgsql_version: 'pg16'
```

#### With Flux Pro

For applications using [Flux Pro](https://fluxui.dev) components:

```yaml
jobs:
  tests:
    uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
      FLUX_USERNAME: ${{ secrets.FLUX_USERNAME }}
      FLUX_PASSWORD: ${{ secrets.FLUX_PASSWORD }}
```

#### Full Example

An application using every available option:

```yaml
jobs:
  tests:
    uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2
    with:
      # Matrix
      php: '["8.3", "8.4"]'
      os: '["ubuntu-24.04"]'
      dependency_version: '["prefer-stable"]'

      # Feature flags
      enable_npm: true
      enable_opensearch: true
      enable_pgsql: true

      # MySQL
      mysql_use_docker: true
      mysql_image: mysql
      mysql_version: '8.0.35'
      mysql_username: root
      mysql_password: root
      mysql_db_name: bisreach
      mysql_db_names: '["contacts", "forage", "tidy", "salutary"]'

      # Node
      node_version_file: '.nvmrc'

      # OpenSearch
      opensearch_version: '2.13.0'
      opensearch_java_opts: '-Xms512m -Xmx512m'

      # PostgreSQL
      pgsql_use_docker: true
      pgsql_image: pgvector/pgvector
      pgsql_version: 'pg16'
      pgsql_username: postgres
      pgsql_password: postgres
      pgsql_db_name: warehouse
      pgsql_db_names: '["analytics"]'

      # PHP
      php_extensions: 'mbstring, dom, fileinfo, mysql, pgsql'
      php_coverage: pcov

      # Redis
      redis_image: valkey/valkey
      redis_version: alpine
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
      FLUX_USERNAME: ${{ secrets.FLUX_USERNAME }}
      FLUX_PASSWORD: ${{ secrets.FLUX_PASSWORD }}
```

#### Workflow Inputs Reference

##### Matrix

| Input                | Type   | Default               | Description                            |
|----------------------|--------|-----------------------|----------------------------------------|
| `php`                | string | `'["8.4"]'`           | JSON array of PHP versions             |
| `os`                 | string | `'["ubuntu-24.04"]'`  | JSON array of runner OS versions       |
| `dependency_version` | string | `'["prefer-stable"]'` | JSON array of Composer stability flags |

##### Feature Flags

| Input               | Type    | Default | Description                      |
|---------------------|---------|---------|----------------------------------|
| `enable_npm`        | boolean | `false` | Enable Node.js install and build |
| `enable_opensearch` | boolean | `false` | Enable OpenSearch container      |
| `enable_pgsql`      | boolean | `false` | Enable PostgreSQL                |

##### MySQL

| Input              | Type    | Default   | Description                                         |
|--------------------|---------|-----------|-----------------------------------------------------|
| `mysql_use_docker` | boolean | `false`   | Use Docker instead of runner service                |
| `mysql_image`      | string  | `mysql`   | Docker image (only with `mysql_use_docker`)         |
| `mysql_version`    | string  | `8.0.32`  | Docker image version (only with `mysql_use_docker`) |
| `mysql_username`   | string  | `root`    | Database username                                   |
| `mysql_password`   | string  | `root`    | Database password                                   |
| `mysql_db_name`    | string  | `laravel` | Primary database name (sets `DB_DATABASE` env)      |
| `mysql_db_names`   | string  | `[]`      | JSON array of additional database names             |

##### Node

| Input               | Type   | Default  | Description                                               |
|---------------------|--------|----------|-----------------------------------------------------------|
| `node_build_script` | string |          | npm script name to run (auto-detects Vite/Mix by default) |
| `node_version_file` | string | `.nvmrc` | Path to node version file                                 |

##### OpenSearch

| Input                  | Type   | Default             | Description          |
|------------------------|--------|---------------------|----------------------|
| `opensearch_version`   | string | `2.13.0`            | Docker image version |
| `opensearch_java_opts` | string | `-Xms512m -Xmx512m` | JVM heap options     |

##### PostgreSQL

| Input              | Type    | Default    | Description                                         |
|--------------------|---------|------------|-----------------------------------------------------|
| `pgsql_use_docker` | boolean | `false`    | Use Docker instead of runner service                |
| `pgsql_image`      | string  | `postgres` | Docker image (only with `pgsql_use_docker`)         |
| `pgsql_version`    | string  | `16`       | Docker image version (only with `pgsql_use_docker`) |
| `pgsql_username`   | string  | `postgres` | Database username                                   |
| `pgsql_password`   | string  | `postgres` | Database password                                   |
| `pgsql_db_name`    | string  | `laravel`  | Primary database name                               |
| `pgsql_db_names`   | string  | `[]`       | JSON array of additional database names             |

##### PHP

| Input            | Type   | Default                          | Description               |
|------------------|--------|----------------------------------|---------------------------|
| `php_extensions` | string | `mbstring, dom, fileinfo, mysql` | PHP extensions to install |
| `php_coverage`   | string | `pcov`                           | Coverage driver           |

##### Redis

| Input           | Type   | Default         | Description          |
|-----------------|--------|-----------------|----------------------|
| `redis_image`   | string | `valkey/valkey` | Docker image         |
| `redis_version` | string | `alpine`        | Docker image version |

##### Secrets

| Secret                          | Required | Description                     |
|---------------------------------|----------|---------------------------------|
| `COMPOSER_OAUTH_GITHUB_ACTIONS` | Yes      | GitHub OAuth token for Composer |
| `FLUX_USERNAME`                 | No       | Flux Pro username               |
| `FLUX_PASSWORD`                 | No       | Flux Pro password               |

#### Schema Imports

The workflow automatically imports Laravel schema dumps when present. Schema files are detected by convention:

**MySQL** looks for these files in order (first match wins):
- `database/schema/mysql.sql`
- `database/schema/mysql-schema.sql`

**PostgreSQL** looks for:
- `database/schema/pgsql.sql`
- `database/schema/pgsql-schema.sql`

**Additional databases** look for schemas named after the database:
- `database/schema/{db_name}.sql`
- `database/schema/{db_name}-schema.sql`

#### Environment Variables

The workflow sets these environment variables automatically for your test suite:

| Variable       | Value                  | Source                 |
|----------------|------------------------|------------------------|
| `DB_DATABASE`  | `mysql_db_name` input  | MySQL primary database |
| `DB_HOST`      | `127.0.0.1`            |                        |
| `DB_PORT`      | `3306`                 |                        |
| `DB_USERNAME`  | `mysql_username` input |                        |
| `DB_PASSWORD`  | `mysql_password` input |                        |
| `REDIS_HOST`   | `127.0.0.1`            |                        |
| `REDIS_PORT`   | `6379`                 |                        |
| `REDIS_SCHEME` | `tcp`                  |                        |
| `APP_URL`      | `http://localhost`     |                        |

#### Runner Service vs Docker

Database actions support two modes:

|                     | Runner Service (default)   | Docker                      |
|---------------------|----------------------------|-----------------------------|
| **Speed**           | Fast (pre-installed)       | Slower (image pull)         |
| **Version control** | Runner's installed version | Exact version pinning       |
| **Use when**        | Version doesn't matter     | You need a specific version |

The runner service starts the pre-installed MySQL/PostgreSQL on the GitHub Actions runner. No image pull is required, which typically saves 20-30 seconds.

#### Pipeline

The workflow optimizes execution by overlapping service startup with language tool installation:

| Phase | Steps                       | Description                                                        |
|-------|-----------------------------|--------------------------------------------------------------------|
| 1     | Start services              | Launch all containers and runner services with `wait: false`       |
| 2     | Install PHP, Composer, Node | Install language tools while services initialize in the background |
| 3     | Finish service setup        | Health checks pass instantly, then databases are created           |

After setup completes, the workflow runs PHPStan static analysis followed by tests.

### Laravel Code Styling

The `code-styling-laravel.yml` workflow automatically fixes code style using [Rector](https://getrector.com/) and [Laravel Pint](https://laravel.com/docs/pint), then commits the changes directly to the branch.

#### Basic Usage

```yaml
# .github/workflows/code-styling.yml
name: Code Styling
on: [push]

jobs:
  styling:
    uses: bisnow/.github/.github/workflows/code-styling-laravel.yml@v2
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

#### With Flux Pro

```yaml
jobs:
  styling:
    uses: bisnow/.github/.github/workflows/code-styling-laravel.yml@v2
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
      FLUX_USERNAME: ${{ secrets.FLUX_USERNAME }}
      FLUX_PASSWORD: ${{ secrets.FLUX_PASSWORD }}
```

#### With a Specific PHP Version

```yaml
jobs:
  styling:
    uses: bisnow/.github/.github/workflows/code-styling-laravel.yml@v2
    with:
      php: '8.3'
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

#### How It Works

1. Checks out the branch that triggered the workflow
2. Installs Composer dependencies (with `--no-scripts` to skip post-install hooks)
3. Runs `composer rector` and commits any changes
4. Runs `composer cs-fix` and commits any changes

Both commits use `[skip actions]` in the message to prevent triggering additional workflow runs.

#### Workflow Inputs Reference

##### Inputs

| Input | Type   | Default | Description |
|-------|--------|---------|-------------|
| `php` | string | `8.4`   | PHP version |

##### Secrets

| Secret                          | Required | Description                     |
|---------------------------------|----------|---------------------------------|
| `COMPOSER_OAUTH_GITHUB_ACTIONS` | Yes      | GitHub OAuth token for Composer |
| `FLUX_USERNAME`                 | No       | Flux Pro username               |
| `FLUX_PASSWORD`                 | No       | Flux Pro password               |

#### Using Both Workflows Together

Most applications will use both workflows. The `bisnow repo:init` command generates this file automatically, but here's what the standard template looks like:

```yaml
# .github/workflows/tests.yml
name: Tests

on:
  push:
    branches-ignore:
      - main

jobs:
  code-styling:
    uses: bisnow/.github/.github/workflows/code-styling-laravel.yml@v2
    with:
      php: '8.4'
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
      FLUX_USERNAME: ${{ secrets.FLUX_USERNAME }}
      FLUX_PASSWORD: ${{ secrets.FLUX_PASSWORD }}

  tests:
    needs: code-styling
    uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2
    with:
      php: '["8.4"]'
      enable_npm: true
      mysql_db_names: '["address", "contacts"]'
    secrets:
      COMPOSER_OAUTH_GITHUB_ACTIONS: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
      FLUX_USERNAME: ${{ secrets.FLUX_USERNAME }}
      FLUX_PASSWORD: ${{ secrets.FLUX_PASSWORD }}

  tests-styling-complete:
    name: Tests and Styling Complete
    needs: tests
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Check if Tests Passed
        run: |
          if [[ "${{ needs.tests.result }}" == "success" ]]; then
            echo "All checks passed"
            exit 0
          else
            echo "Some checks failed"
            echo "tests: ${{ needs.tests.result }}"
            exit 1
          fi
```

The `tests-styling-complete` job acts as a single status check for branch protection rules, so you only need to require one check instead of two.

## Composite Actions

Each action can be used independently in your own workflows if the reusable workflow doesn't fit your needs.

### setup-mysql

Starts MySQL, creates databases, and imports schema dumps.

```yaml
- uses: bisnow/.github/.github/actions/setup-mysql@v2
  with:
    db-name: my_app
    db-names: '["my_app_testing", "my_app_audit"]'
```

#### Inputs

| Input            | Default      | Description                                    |
|------------------|--------------|------------------------------------------------|
| `wait`           | `true`       | Wait for service to be ready before continuing |
| `use-docker`     | `false`      | Use Docker instead of runner service           |
| `image`          | `mysql`      | Docker image                                   |
| `version`        | `8.0.32`     | Docker image version                           |
| `container-name` | `mysql-test` | Docker container name                          |
| `host`           | `127.0.0.1`  | Database host                                  |
| `port`           | `3306`       | Database port                                  |
| `username`       | `root`       | Database username                              |
| `password`       | `root`       | Database password                              |
| `db-name`        | `laravel`    | Primary database name                          |
| `db-names`       | `[]`         | JSON array of additional databases             |

### setup-opensearch

Starts an OpenSearch container configured for testing (single-node, security disabled).

```yaml
- uses: bisnow/.github/.github/actions/setup-opensearch@v2
```

#### Inputs

| Input            | Default             | Description                                    |
|------------------|---------------------|------------------------------------------------|
| `wait`           | `true`              | Wait for service to be ready before continuing |
| `version`        | `2.13.0`            | Docker image version                           |
| `container-name` | `opensearch-test`   | Docker container name                          |
| `port`           | `9200`              | Port                                           |
| `java-opts`      | `-Xms512m -Xmx512m` | JVM heap options                               |

### setup-pgsql

Starts PostgreSQL, creates a user, creates databases, and imports schema dumps.

```yaml
- uses: bisnow/.github/.github/actions/setup-pgsql@v2
  with:
    username: my_user
    password: my_password
    db-name: my_app
```

With a custom Docker image (e.g., pgvector):

```yaml
- uses: bisnow/.github/.github/actions/setup-pgsql@v2
  with:
    use-docker: true
    image: pgvector/pgvector
    version: pg16
    db-name: my_app
```

#### Inputs

| Input            | Default      | Description                                    |
|------------------|--------------|------------------------------------------------|
| `wait`           | `true`       | Wait for service to be ready before continuing |
| `use-docker`     | `false`      | Use Docker instead of runner service           |
| `image`          | `postgres`   | Docker image                                   |
| `version`        | `16`         | Docker image version                           |
| `container-name` | `pgsql-test` | Docker container name                          |
| `host`           | `127.0.0.1`  | Database host                                  |
| `port`           | `5432`       | Database port                                  |
| `username`       | `postgres`   | Database username                              |
| `password`       | `postgres`   | Database password                              |
| `db-name`        | `laravel`    | Primary database name                          |
| `db-names`       | `[]`         | JSON array of additional databases             |

### setup-php-composer

Sets up PHP with extensions, configures Composer authentication, caches dependencies, and installs packages.

```yaml
- uses: bisnow/.github/.github/actions/setup-php-composer@v2
  with:
    php-version: '8.4'
    composer-oauth-token: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

With Flux Pro and additional extensions:

```yaml
- uses: bisnow/.github/.github/actions/setup-php-composer@v2
  with:
    php-version: '8.4'
    extensions: 'mbstring, dom, fileinfo, mysql, pgsql, redis'
    composer-oauth-token: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
    flux-username: ${{ secrets.FLUX_USERNAME }}
    flux-password: ${{ secrets.FLUX_PASSWORD }}
```

#### Inputs

| Input                  | Required | Default                          | Description        |
|------------------------|----------|----------------------------------|--------------------|
| `php-version`          | Yes      |                                  | PHP version        |
| `extensions`           | No       | `mbstring, dom, fileinfo, mysql` | PHP extensions     |
| `coverage`             | No       | `pcov`                           | Coverage driver    |
| `composer-oauth-token` | Yes      |                                  | GitHub OAuth token |
| `flux-username`        | No       |                                  | Flux Pro username  |
| `flux-password`        | No       |                                  | Flux Pro password  |

### setup-node-npm

Sets up Node.js (version from file), caches dependencies, installs packages, and builds assets. The build step automatically detects the bundler:

- **Vite** (`vite.config.js` or `vite.config.ts`) runs `npm run build`
- **Laravel Mix** (`webpack.mix.js`) runs `npm run prod`
- **No bundler detected** fails with an error suggesting you set `build-script`

```yaml
- uses: bisnow/.github/.github/actions/setup-node-npm@v2
```

With an explicit build script:

```yaml
- uses: bisnow/.github/.github/actions/setup-node-npm@v2
  with:
    build-script: prod
```

#### Inputs

| Input               | Default  | Description                                               |
|---------------------|----------|-----------------------------------------------------------|
| `build-script`      |          | npm script name to run (auto-detects Vite/Mix by default) |
| `node-version-file` | `.nvmrc` | Path to node version file                                 |

### setup-redis

Starts a Redis-compatible container.

```yaml
- uses: bisnow/.github/.github/actions/setup-redis@v2
```

With a specific Redis image:

```yaml
- uses: bisnow/.github/.github/actions/setup-redis@v2
  with:
    image: redis
    version: '7-alpine'
```

#### Inputs

| Input            | Default         | Description                                    |
|------------------|-----------------|------------------------------------------------|
| `wait`           | `true`          | Wait for service to be ready before continuing |
| `image`          | `valkey/valkey` | Docker image                                   |
| `version`        | `alpine`        | Docker image version                           |
| `container-name` | `redis-test`    | Docker container name                          |
| `port`           | `6379`          | Port                                           |

## Shared Configuration

### Pull Request Templates

This repository provides two PR templates that are automatically available to all repositories in the Bisnow organization:

- **[Default PR Template](.github/PULL_REQUEST_TEMPLATE.md)** - Used for feature branches. Includes sections for Jira ticket, release notes, description, testing instructions, QA, dependencies, and deployment notes.
- **[Release PR Template](.github/PULL_REQUEST_TEMPLATE/release.md)** - Used for release PRs. Includes sections for related PRs, release notes, dependencies, and deployment notes.

To use the release template, append `?template=release.md` to the PR creation URL, or select it from the template dropdown when creating a PR on GitHub.

### EditorConfig

The `.editorconfig` file defines consistent coding styles across editors and IDEs:

- UTF-8 charset, LF line endings, trailing newline
- 4-space indentation for most files
- 2-space indentation for YAML files (except `docker-compose.yml` which uses 4)
- No trailing whitespace trimming in Markdown files

This file is automatically picked up by editors that support [EditorConfig](https://editorconfig.org/).

## Versioning

This repository follows [semantic versioning](https://semver.org/). Reference a major version tag for automatic patch updates:

```yaml
# Recommended: major version tag (gets patches automatically)
uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2

# Exact version (pinned, no automatic updates)
uses: bisnow/.github/.github/workflows/tests-laravel.yml@v2.0.0
```

**Breaking changes** (new major version) include renaming inputs, removing inputs, or changing default behavior. Non-breaking additions (new optional inputs, new actions) are released as minor versions.
