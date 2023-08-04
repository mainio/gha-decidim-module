# Decidim GitHub actions

This module provides centralized workflow configurations for
[Decidim](https://decidim.org/) modules that we can use for centralizing the
configurations for the workflows and maintain them in a single location.

Each shared action has a dedicated folder in this repository where the actions
can be referred from. This repository also contains shared ready-made workflow
configurations to be used within the modules for easier management.

The workflows that this repository provides:

- `build_app.yml` - Takes care of building the test application for easier
  reuse in case the target repository has multiple modules in it.
- `ci_js.yml` - Contains the workflow for running `npm test` within the module's
  repository, usually using Jest within the Decidim context.
- `ci_rspec.yml` - Contains the workflow for running `rspec` within the module's
  repository, using RSpec.
- `lint.yml` - Lints the code using different linters, such as Rubocop,
  Stylelint and ESLint.

## Usage

This repository contains both GitHub actions as well as shared workflow
configuration files. The actions can be used individually but the shared
workflow files allow us to maintain some of the worker and service
configurations in a centralized locations. This makes it easy e.g. to update the
PostgreSQL versions when needed or add additional services to the list.

### Shared workflow files

The recommended way is to run the workflows using the shared workflow
configuration files located at the `.github/workflows` directory of this
repository. The following sections give examples on how to configure these
properly for your module repository.

For all configured workflows, the following configurations are common:

- `on` flag - Adds the correct triggers when these workflows are run (e.g.
  during pushes into specific branches or during pull requests)
- `concurrency` flag - Disables concurrent workflows using the
  [`concurrency` flag](https://docs.github.com/en/actions/using-jobs/using-concurrency).
  E.g. when you have pull requests open that have actions running on previous
  commits, this would cancel the previous runs when new commits are pushed to
  the same branch.

#### Runnin RSpec

To run the RSpec tests, configure the following workflow file within your
module's repository:

`.github/workflows/ci_decidim_{MODULE_NAME}.yml`
(replace `{MODULE_NAME}` with the name of your module)

The contents of this file should look as follows:

```yml
# Replace the {MODULE_NAME} below with the actual name of the module.
name: "[CI] {MODULE_NAME}"
on:
  push:
    branches:
      - develop
      - main
      - release/*
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  build_app:
    name: Build test application
    uses: mainio/gha-decidim-module/.github/workflows/build_app.yml@main
    secrets: inherit
  main:
    needs: build_app
    name: Tests
    uses: mainio/gha-decidim-module/.github/workflows/ci_rspec.yml@main
    secrets: inherit
```

The jobs of this workflow will handle the following tasks:

- `build_app` - Builds the test application, i.e. `spec/decidim_dummy_app` and
  stores it in a cache. Pre-building the app will speed up consequent runs on
  the same workflow, e.g. if the workflow fails due to a flaky test.
- `main` - Loads the cached application and runs the `rspec` tests within the
  repository using that application.

#### Running JS tests (e.g. Jest)

> [!NOTE]
> If you have not implemented any Jest tests for the repository, do not add this
> workflow.

To run the JS tests (i.e. `npm test`, usually using Jest), configure the
following workflow file within your module's repository:

`.github/workflows/ci_decidim_{MODULE_NAME}_js.yml`
(replace `{MODULE_NAME}` with the name of your module)

The contents of this file should look as follows:

```yml
# Replace the {MODULE_NAME} below with the actual name of the module.
name: "[CI] {MODULE_NAME} JavaScript"
on:
  push:
    branches:
      - develop
      - main
      - release/*
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  main:
    name: JS tests
    uses: mainio/gha-decidim-module/.github/workflows/ci_js.yml@main
    secrets: inherit
```

The `main` job takes care of installing the NPM dependencies and running
`npm test` within the repository.

#### Linting

To run the different linters (Rubocop, Stylelint and ESLint), configure the
following workflow file within your module's repository:

```yml
name: "[CI] Lint"
on:
  push:
    branches:
      - develop
      - main
      - release/*
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  lint:
    name: Lint code
    uses: mainio/gha-decidim-module/.github/workflows/lint.yml@main
    secrets: inherit
    with:
      eslint: true # Remove this if you do not have ESLint configured in the repository
      stylelint: true # Remove this if you do not have Stylelint configured in the repository
```

The `lint` job takes care of installing the necessary requirements for running
the different linters as well as running the linting tasks. The example
configuration runs all the linters (Rubocop, ESLint and Stylelint) but you can
enable or disable the different linters using these input options:

- `rubocop` (boolean, default: `true`) - Defines whether Rubocop will be run
- `eslint` (boolean, default: `false`) - Defines whether ESLint will be run
- `stylelint` (boolean, default: `false`) - Defines whether Stylelint will be
  run

> [!NOTE]
> Before enabling ESLint or Stylelint, please make sure that the commands
> `npm run lint` and `npm run stylelint` are defined and work correctly within
> the repository.

### Actions

If you need more granular control regarding the runner environment and attached
services, you can also use the actions directly that are available in the
subfolders of this repository. The following sections explain each action in
more detail.

#### Setup app (`setup_app`)

> [!NOTE]
> This action should be used in conjunction with the `test_rspec` action for it
> to be useful. This only generates the test application which is why it may not
> be very useful otherwise.

The `setup_app` action takes care of generating the test application (i.e.
`spec/decidim_dummy_app`) and storing it in the GitHub actions cache for quick
usage within your actual jobs. This is a separate action in order to make it
quicker to run the test jobs consequently in case they fail due to a flaky test.

To use this action within your workflow, configure the following workflow file
within your module's repository:

```yml
name: "[CI] Setup"
on:
  push:
    branches:
      - develop
      - main
      - release/*
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  build_app:
    name: Build test application
    uses: mainio/gha-decidim-module/setup_app@main
```

The action provides the following input options:

- `use-cached-app` (boolean, default: `true`) - Defines whether the generated
  test application is stored in a cache.
    * This is required in case you need to use the same application across
      multiple jobs which is why it is enabled by default.

#### Test RSpec (`test_rspec`)

The `test_rspec` action runs the RSpec tests within the module's repository.
Prior to using this action, you need to have generated the test application
(see `setup_app` for more details) in order for the test application to exist.

To use this action within your workflow, configure the following workflow file
within your module's repository:

```yml
name: "[CI] RSpec"
on:
  push:
    branches:
      - develop
      - main
      - release/*
  pull_request:

env:
  CI: "true"
  CODECOV: "true"

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  main:
    name: Tests
    runs-on: ubuntu-latest
    timeout-minutes: 60
    services:
      postgres:
        image: postgres:11
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        env:
          POSTGRES_PASSWORD: postgres
    env:
      DATABASE_USERNAME: postgres
      DATABASE_PASSWORD: postgres
      DATABASE_HOST: localhost
    steps:
      - uses: mainio/gha-decidim-module/setup_app@main
        with:
          use-cached-app: false
      - uses: mainio/gha-decidim-module/test_rspec@main
        with:
          use-cached-app: false
```

The action provides the following input options:

- `use-cached-app` (boolean, default: `true`) - Defines whether the generated
  test application is used to run the specs. If used, please also define the
  `build_app` action separately in a separate job.
  * Note that when this configuration is used, also the database schema will be
    loaded prior to running the tests regardless of the `load-schema`
    configuration.
- `load-schema` (boolean, default: `true`) - Defines whether the test database
  is purged and the database schema is re-loaded before running the tests.

#### Test JavaScript (`test_js`)

The `test_js` action runs the JavaScript tests within the module's repository
using `npm test`.

To use this action within your workflow, configure the following workflow file
within your module's repository:

```yml
name: "[CI] JS"
on:
  push:
    branches:
      - develop
      - main
      - release/*
  pull_request:

env:
  CI: "true"
  CODECOV: "true"

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  main:
    name: Tests
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - uses: mainio/gha-decidim-module/test_js@main
        with:
          use-cached-app: false
```

Note that before configuring this action, you should ensure that you have an NPM
script named `test` defined in the module's `package.json` file and running the
command `npm run test` succeeds without errors.

#### Lint (`lint`)

The `lint` action configures and runs the linters (Rubocop, ESLint and
Stylelint) for your module's repository. By default, only Rubocop is run but if
you want to enable ESLint and/or Stylelint, you can do so by using the input
options (see below).

To use this action within your workflow, configure the following workflow file
within your module's repository:

```yml
name: "[CI] Lint"
on:
  push:
    branches:
      - develop
      - main
      - release/*
  pull_request:

env:
  CI: "true"
  CODECOV: "true"

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  main:
    name: Tests
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - uses: mainio/gha-decidim-module/lint@main
        with:
          eslint: true
          stylelint: true
```

Note that before enabling either the `eslint` or `stylelint` options, you should
ensure that you have an NPM scripts named `lint` and `stylelint` defined in the
module's `package.json` file and running the command `npm run lint` and
`npm run stylelint` succeed without errors.

The action provides the following input options:

- `rubocop` (boolean, default: `true`) - Defines whether the Rubocop will be
  run.
- `eslint` (boolean, default: `false`) - Defines whether ESLint will be run
  using `npm run lint`.
  * Before enabling this option, make sure that `npm run lint` succeeds without
    errors within your module's repository.
- `stylelint` (boolean, default: `false`) - Defines whether Stylelint will be
  run using `npm run lint`.
  * Before enabling this option, make sure that `npm run stylelint` succeeds
    without errors within your module's repository.

## Contributing

We are welcome for contributions for these actions and shared workflow
configurations but we use them ourselves in all of our modules. Therefore, they
may be slightly opinionated towards the needs of the modules we are maintaining,
although they are following similar conventions and patterns from the Decidim
core.

Before introducing any new features or adding new services to the actions or
workflow configurations, please
[open an issue](https://github.com/mainio/gha-decidim-module/issues/new) first
and briefly and clearly explain what you would like to change and why. We can
then reach a conclusion about the change before breaking all the workflows out
there.

In case you are fixing a bug, those contributions are of course always welcome.

## License

This repository is distributed under the GNU AFFERO GENERAL PUBLIC LICENSE.

For further details, see the [license](LICENSE-AGPLv3.txt).
