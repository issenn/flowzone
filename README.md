# Flowzone

![ridiculous logo about hating and flowing](logo.png)

Reusable, opinionated, zero-conf workflows for GitHub actions

## Contents

- [Getting Started](#getting-started)
- [Usage](#usage)
  - [Commit Message Commands](#commit-message-commands)
    - [Skipping Workflow Runs](#skipping-workflow-runs)
- [Supported project types](#supported-project-types)
  - [npm](#npm)
  - [Docker](#docker)
  - [balena](#balena)
  - [Custom](#custom)
  - [Versioning](#versioning)
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
    - [`runs_on`](#runs_on)
    - [`jobs_timeout_minutes`](#jobs_timeout_minutes)
    - [`working_directory`](#working_directory)
    - [`docker_images`](#docker_images)
    - [`balena_environment`](#balena_environment)
    - [`balena_slugs`](#balena_slugs)
    - [`protect_branch`](#protect_branch)
    - [`required_approving_review_count`](#required_approving_review_count)
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

### Merging

Flowzone only supports merging with a merge commit. This option is shown as **Merge pull request** on an in-progress PR.

Avoid using **Squash and merge** or **Rebase and merge** as these methods will result in a new commit SHA not matching anything from the original PR.
This would prevent Flowzone from finding and finalizing existing draft artifacts.

You can read more about the available merge methods [here](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/about-merge-methods-on-github).

### Commit Message

#### Skipping Workflow Runs

Workflows that would otherwise be triggered using `on: push` or `on: pull_request` won't be triggered if you add any of the following strings to the commit message in a push, or the HEAD commit of a pull request:

```text
[skip ci]
[ci skip]
[no ci]
[skip actions]
[actions skip]
```

Alternatively, you can end the commit message with two empty lines followed by either:

```text
skip-checks:true
skip-checks: true
```

You won't be able to merge the pull request if your repository is configured to require specific checks to pass first. To allow the pull request to be merged you can push a new commit to the pull request without the skip instruction in the commit message.

For further reading please check [Github Skipping Workflow Documentation](https://docs.github.com/en/actions/managing-workflow-runs/skipping-workflow-runs).

## Supported project types

Note that these project types are _not_ mutually exclusive, and your project may execute one or more of the following.

### npm

These tests will be run if a `package.json` file is found in the root of the repository.
If a `package-lock.json` file is found in the root of the repository, dependencies will be installed with `npm ci`, otherwise `npm i` will be used.
If a build script is present in `package.json` it will be called before the tests are run. Testing is done by calling `npm test`.

Multiple LTS versions of Node.js will be tested as long as they meet the [range](https://github.com/npm/node-semver#ranges) defined in [engines](https://docs.npmjs.com/cli/v8/configuring-npm/package-json#engines) in `package.json`.
Node.js 16.x will be used for testing if engines are not defined.

To disable publishing of artifacts set `"private": true` in `package.json`.

### Docker

If a `docker-compose.yml` _and_ a `docker-compose.test.yml` are found in the root of the repository, Flowzone will test your project using Docker compose.

The compose script will merge the to yaml file together and wait on a container named `sut` to finish. The result of the test is determined by the exit code of the `sut` container.
Typically, the `sut` container will execute your e2e or integration tests and will exit on test completion.

If you need to provide environment variables to the compose environment you can add a repository secret called [`COMPOSE_VARS`](#compose_vars) that should be a base64 encoded `.env` file.
This will be decoded and written to a `.env` file inside the test worker at runtime.

To enable publishing of Docker artifacts set [`docker_images`](#docker_images) input to correct value of docker image repositories without tags eg  - eg `ghcr.io/${{ github.repository }}`.

For advanced Docker build options, including multi-arch, add a [docker-bake.hcl](https://docs.docker.com/engine/reference/commandline/buildx_bake/) file to your project.

```hcl
// docker-bake.hcl
// https://github.com/docker/metadata-action#bake-definition
target "docker-metadata-action" {}

target "build" {
  inherits = ["docker-metadata-action"]
  context = "./"
  dockerfile = "Dockerfile"
  args = {
    foo = "bar"
  }
  platforms = [
    "linux/amd64",
    "linux/arm/v7",
    "linux/arm64",
  ]
}
```

### balena

If a `balena.yml` file is found in the root of the repository Flowzone will attempt to push draft releases to your application slug(s) and finalize on merge.

To disable publishing of releases to balenaCloud set [`balena_slugs`](#balena_slugs) to `""`.

## Custom

_Note: Custom Flowzone actions are not a guaranteed stable interface and should be merged to Flowzone core when possible._

If any [composite actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) are found at the following paths
they will be executed automatically at the described stages of the workflow.

```bash
.github
├── actions
│   ├── clean
│   │   └── action.yml
│   ├── finalize
│   │   └── action.yml
│   ├── publish
│   │   └── action.yml
│   └── test
│       └── action.yml
└── workflows
    ├── flowzone.yml
```

A `test` action will run in parallel to any other supported project tests. See [test/action.yml](.github/actions/test/action.yml) for an example.

A `publish` action will run after all the supported tests have completed. See [publish/action.yml](.github/actions/publish/action.yml) for an example.

A `finalize` action will run after a pull request is merged. See [finalize/action.yml](.github/actions/finalize/action.yml) for an example.

A `clean` action will run after a pull request is closed (including merged). See [clean/action.yml](.github/actions/clean/action.yml) for an example.

More interfaces may be added in the future. Open an issue if you have a use case that is not covered!

### Versioning

If a `repo.yml` file is found in the root of the repository Flowzone will attempt run [balena-versionist](https://github.com/product-os/balena-versionist) directly on the PR's source.

If you have [VersionBot3](https://github.com/apps/versionbot3) installed it will be ignored as far as versioning is concerned, so no need to disable it.

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

Required for [versioned](#versioning) projects.

#### `GPG_PASSPHRASE`

Passphrase to decrypt GPG private key.

Required for [versioned](#versioning) projects.

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

#### `runs_on`

GitHub actions runner (e.g. `[\"self-hosted\",\"linux\",\"x64\"]` for self-hosted runners.)

Type: _string_

Default: `[\"ubuntu-latest\"]`

#### `jobs_timeout_minutes`

Job(s) timeout.

Type: _number_

Default: `360`

#### `working_directory`

GitHub actions working directory.

Type: _string_

Default: `.`

#### `docker_images`

Comma-delimited string of Docker images (without tags) to publish (skipped if empty).

Type: _string_

Default: `''`

#### `balena_environment`

Alternative balenaCloud environment (e.g. balena-staging.com)

Type: _string_

Default: `balena-cloud.com`

#### `balena_slugs`

Comma-delimited string of balenaCloud apps, fleets, or blocks to deploy (skipped if empty).

Type: _string_

Default: `${{ github.repository }}`

#### `protect_branch`

Set to false to disable updating branch protection rules after a successful run.

Type: _boolean_

Default: `true`

#### `required_approving_review_count`

Setting this value to zero effectively means merge==deploy without approval(s).

Type: _string_

Default: `1`

#### `npm_test_command`

Setting this value to specify a different command when initiating the npm tests.

Type: _string_

Default: `npm test`


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
