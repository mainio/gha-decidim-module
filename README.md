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
