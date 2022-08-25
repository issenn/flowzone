# Flowzone

Reusable, opinionated, zero-conf workflows for GitHub actions

## Contents

- [Getting Started](#getting-started)
- [Usage](#usage)
- [Supported project types](#supported-project-types)
  - [npm](#npm)
  - [Docker](#docker)
  - [balena](#balena)
  - [Versionbot](#versionbot)
- [Customization](#customization)
  - [Secrets](#secrets)
    - [`FLOWZONE_TOKEN`](#flowzone_token)
    - [`GPG_PRIVATE_KEY`](#gpg_private_key)
    - [`GPG_PASSPHRASE`](#gpg_passphrase)
    - [`NPM_TOKEN`](#npm_token)
    - [`GHCR_TOKEN`](#ghcr_token)
    - [`DOCKERHUB_USER`](#dockerhub_user)
    - [`DOCKERHUB_TOKEN`](#dockerhub_token)
    - [`BALENA_API_KEY`](#balena_api_key)
    - [`COMPOSE_VARS`](#compose_vars)
  - [Inputs](#inputs)
    - [`working_directory`](#working_directory)
    - [`dockerhub_repo`](#dockerhub_repo)
    - [`ghcr_repo`](#ghcr_repo)
    - [`balena_slugs`](#balena_slugs)
    - [`npm_registry`](#npm_registry)
    - [`docker_platforms`](#docker_platforms)
    - [`docker_context`](#docker_context)
    - [`docker_file`](#docker_file)
    - [`docker_target`](#docker_target)
    - [`node_versions`](#node_versions)
    - [`skip_versioning`](#skip_versioning)
    - [`protect_branch`](#protect_branch)
    - [`required_approving_review_count`](#required_approving_review_count)
    - [`required_status_checks`](#required_status_checks)
    - [`verbose`](#-verbose-)
- [Maintenance](#maintenance)
  - [Generate GPG keys](#generate-gpg-keys)
- [Help](#help)
- [Contributing](#contributing)

## Getting Started

Open a PR with the following changes to test and enable Flowzone:

1. Create `.github/workflows/flowzone.yml` (see [Usage](#usage)) in a new PR
2. Set the input `protect_branch: false` until you are certain that the tests are passing
3. Ensure your `package.json`, `docker-compose.test.yml`, `balena.yml`, etc. contain correct information and Flowzone is passing all tests
4. Remove the `protect_branch` input to apply new rules automatically. This requires admin access to revert!
5. Seek approval or self-certify!

## Usage

Simply add the following to `./github/workflows/flowzone.yml`:

```yml
name: Flowzone

on:
  pull_request:
    types: [opened, synchronize, closed]
    branches:
      - "main"
      - "master"

jobs:
  flowzone:
    name: Flowzone
    uses: product-os/flowzone/.github/workflows/flowzone.yml@master
    secrets:
      FLOWZONE_TOKEN: ${{ secrets.FLOWZONE_TOKEN }}
      GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
      GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
      DOCKERHUB_USER: ${{ secrets.DOCKER_REGISTRY_USER }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKER_REGISTRY_PASS }}
      BALENA_API_KEY: ${{ secrets.BALENA_API_KEY }}
    with:
      protect_branch: false
```

Workflows that call reusable workflows in the same organization or enterprise can use the inherit keyword to implicitly pass the secrets.

```yml
name: Flowzone

on:
  pull_request:
    types: [opened, synchronize, closed]
    branches:
      - "main"
      - "master"

jobs:
  flowzone:
    name: Flowzone
    uses: product-os/flowzone/.github/workflows/flowzone.yml@master
    secrets: inherit
```

Flowzone will automatically select an appropriate runner based on your project's code.

## Supported project types

Note that these project types are _not_ mutually exclusive, and your project may execute one or more of the following.

### npm

These tests will be run if a `package.json` file is found in the root of the repository. If a `package-lock.json` file is found in the root of the repository, dependencies will be install with `npm ci`, otherwise `npm i` will be used.
If a build script is present in `package.json` it will be called before the tests are run. Testing is done by calling `npm test`.

The [`node_versions`](#node_versions) will determine the Node.js versions used for testing.

To disable publishing of artifacts set [`npm_registry`](#npm_registry) to `""` or set `"private": true` in `package.json`.

### Docker

If a `docker-compose.yml` _and_ a `docker-compose.test.yml` are found in the root of the repository, Flowzone will test your project using Docker compose.

The compose script will merge the to yaml file together and wait on a container named `sut` to finish. The result of the test is determined by the exit code of the `sut` container.
Typically, the `sut` container will execute your e2e or integration tests and will exit on test completion.

If you need to provide environment variables to the compose environment you can add a repository secret called [`COMPOSE_VARS`](#compose_vars) that should be a base64 encoded `.env` file.
This will be decoded and written to a `.env` file inside the test worker at runtime.

To disable publishing of artifacts to Docker Hub set [`dockerhub_repo`](#dockerhub_repo) to `""`.

To disable publishing of artifacts to GitHub Container Registry set [`ghcr_repo`](#ghcr_repo) to `""`.

### balena

If a `balena.yml` file is found in the root of the repository Flowzone will attempt to push draft releases to your application slug(s) and finalize on merge.

To disable publishing of releases to balenaCloud set [`balena_slugs`](#balena_slugs) to `""`.

### Versionbot

Flowzone ignores [versionbot](https://github.com/apps/versionbot3) and will run [balena-versionist](https://github.com/product-os/balena-versionist) directly on the PR's branch.

## Customization

### Secrets

The following secrets should be set by an Owner at the Organization level,
but they can also be [configured for personal repositories](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository).

These secrets can also be found at the top of [flowzone.yml](./.github/workflows/flowzone.yml).

#### `FLOWZONE_TOKEN`

Personal access token (PAT) for the GitHub service account with admin/owner permissions.

Always required.

#### `GPG_PRIVATE_KEY`

GPG private key exported with `gpg --armor --export-secret-keys ...` to sign commits.

Required if [versioning is enabled](#skip_versioning).

#### `GPG_PASSPHRASE`

Passphrase to decrypt GPG private key.

Required if [versioning is enabled](#skip_versioning).

#### `NPM_TOKEN`

The npm auth token to use for publishing.

Required for [npm](#npm) projects.

#### `GHCR_TOKEN`

A personal access token to publish to the GitHub Container Registry, will use [`FLOWZONE_TOKEN`](#flowzone_token) if unset.

Required for [Docker](#docker) projects.

#### `DOCKERHUB_USER`

Username to publish to the Docker Hub container registry.

Required for [Docker](#docker) projects.

#### `DOCKERHUB_TOKEN`

A [personal access token](https://docs.docker.com/docker-hub/access-tokens/) to publish to the Docker Hub container registry.

Required for [Docker](#docker) projects.

#### `BALENA_API_KEY`

[API key](https://www.balena.io/docs/learn/manage/account/#api-keys) for pushing releases to balena applications.

Required for [balena](#balena) projects.

#### `COMPOSE_VARS`

Optional base64 encoded docker-compose `.env` file for testing [Docker](#docker) projects.

### Inputs

These inputs are all optional and include some opinionated defaults.
They can also be found at the top of [flowzone.yml](./.github/workflows/flowzone.yml).

#### `working_directory`

GitHub actions working directory.

Type: _string_

Default: `.`

#### `dockerhub_repo`

Docker Hub repository for Docker projects (skipped if empty).

Type: _string_

Default: `${{ github.repository }}`

#### `ghcr_repo`

GitHub Container Registry repository for Docker projects (skipped if empty).

Type: _string_

Default: `${{ github.repository }}`

#### `balena_slugs`

Comma-delimited string of balenaCloud apps, fleets, or blocks to deploy (skipped if empty).

Type: _string_

Default: `${{ github.repository }}`

#### `npm_registry`

Registry for publishing npm projects (skipped if empty).

Type: _string_

Default: `registry.npmjs.org`

#### `docker_platforms`

Comma-delimited string of Docker target platforms.

Type: _string_

Default: `linux/amd64,linux/arm64,linux/arm/v7`

#### `docker_context`

Docker build context directory relative to [`working_directory`](#working_directory).

Type: _string_

Default: `.`

#### `docker_file`

Path to the Dockerfile relative to the context.

Type: _string_

Default: `Dockerfile`

#### `docker_target`

Sets the target stage to build.

Type: _string_

Default: all stages

#### `node_versions`

Comma-delimited string of Node.js versions to test.

Type: _string_

Default: `14.x,16.x,18.x`

#### `skip_versioning`

Set to `true` to skip adding a version commit on top of the original source.

Type: _boolean_

Default: `false`

#### `protect_branch`

Set to false to disable updating branch protection rules after a successful run.

Type: _boolean_

Default: `true`

#### `required_approving_review_count`

Setting this value to zero effectively means merge==deploy without approval(s).

Type: _string_

Default: `1`

#### `required_status_checks`

Comma-delimited string of additional required status checks for branch protection.

Type: _string_

Default: `"Flowzone / npm publish","Flowzone / docker publish","Flowzone / balena all"`

#### `verbose`

Enable shell command tracing.

Type: _boolean_

Default: `false`

## Maintenance

### Generate GPG keys

[Managing commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification)

1. generate new GPG key signing ensuring the name matches an existing GitHub user/identity

   ```bash
   gpg --full-generate-key
   ```

2. get the key id

   ```bash
   gpg --list-secret-keys --keyid-format=long
   ```

3. export the key to be stored in `GPG_PRIVATE_KEY` GitHub organisation secret

   ```bash
   gpg --armor --export-secret-keys {{secret_key_id}}
   ```

4. set `GPG_PASSPHRASE` and `GPG_PRIVATE_KEY` GitHub organisation secrets

## Help

If you're having trouble getting the project running,
submit an issue or post on the forums at <https://forums.balena.io>.

## Contributing

Please open an issue or submit a pull request with any features, fixes, or changes.
