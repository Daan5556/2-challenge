# CI pipeline

**Author:** Daan Eggen  
**Date:** 23/05/2026  
**Version:** 1.0

---

To keep the code style consistent in our project, I added a CI pipeline. CI
stands for Continuous Integration. This means that Github automatically runs a
set of checks when code is pushed, or when a pull request is created.

For this project, the pipeline is used to enforce a shared formatting style. The
project uses the TALL stack, so this can be extended with PHP and Laravel tools
like Laravel Pint, PHPUnit or Pest, and PHPStan or Larastan. When a developer
creates a pull request, Github Actions can check if the code matches the agreed
style and quality rules. If this is not the case, the pull request will show a
failed check and the developer knows what needs to be fixed before merging.

## Pipeline script

The pipeline configuration is stored in `docs/devops/ci.yml`:

```yml
name: CI

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6.0.2
      - name: Prettier check
        run: npx prettier --check .
```

This pipeline runs on every push to the `main` branch, and on every pull request
that targets `main`. It can also be started manually with `workflow_dispatch`.
The build job runs on an Ubuntu runner, checks out the repository, and then runs
the style check.

The command:

```bash
npx prettier --check .
```

checks all files in the repository against Prettier. It does not change files by
itself, but it reports which files are not formatted correctly. This is useful
because it keeps the responsibility clear: developers still make the change, but
Github makes sure the agreed style is not forgotten.

In a TALL project, the same role can be handled for PHP code by Laravel Pint.
That check would fail the pipeline when PHP files do not follow the configured
Laravel style.

## Enforcing style

Without an automated check, every developer has to remember to run formatting
manually. This can easily be forgotten, especially when working on a deadline or
when switching between tasks. By placing the formatting check in the pipeline,
the project does not rely on memory or manual discipline only.

This saves time during code review. Reviewers do not have to spend time
commenting on spacing, indentation or line wrapping. The pipeline handles this
automatically, so the team can focus the review on implementation details,
architecture and correctness.

## Extending code quality

The current pipeline only checks formatting, but the same setup can be extended
to guard more parts of the project quality. For a PHP TALL project, we can add
steps for:

- installing Composer dependencies
- checking PHP style with Laravel Pint
- running automated tests with PHPUnit or Pest
- running static analysis with PHPStan or Larastan
- checking security or dependency problems

An extended version could look like this:

```yml
      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist
      - name: Pint check
        run: ./vendor/bin/pint --test
      - name: Test
        run: php artisan test
      - name: Static analysis
        run: ./vendor/bin/phpstan analyse
```

When these steps are added, every pull request gets tested in the same way. This
reduces the chance that developers forget to run tests or linting locally. It
also prevents losing time later in the process, because problems are found
directly when the code is submitted.

## Conclusion

With this CI pipeline in place, the project has an automatic quality gate before
code is merged. At the moment it enforces a consistent code style. In the
future, the same pipeline can be expanded with testing, linting, static analysis
and dependency checks. This makes the development process more reliable and
gives developers quick feedback without requiring them to remember every manual
command.
