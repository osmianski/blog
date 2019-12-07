# Organizing PHP Projects And Packages

In my recent PHP projects, in addition to developing project-specific source-code, I actively develop several library packages in `vendor` directory, which I then use in another projects. I also typically hotfix already deployed code while developing new features at the same time.

After organizing directories, Git repositories and workflow ad-hoc for quite some time, I think now it's time to reflect and document some "standard" way of doing that. It is also useful to document solutions to the certain problems I constantly run into.

That's what this article is about.

This article assumes a basic knowledge of [Git](https://git-scm.com/), [Composer](https://getcomposer.org/) and [semantic versioning](https://semver.org/).

Contents:

{{ toc }}

## Git Repositories

Projects and library packages are stored in Git repositories on GitHub or BitBucket. Repository names should match Composer package names (in the `name` property of `composer.json` file).

## Long-Living Branches

These Git repositories have the following long-living branches:

* `master` - for developing next big version; next big version is released in big chunks, from 1.5 months to a year each.
* `vX` branches (`v1`, `v2`, ...) - for supporting `vX.Y.Z` versions; these branches are created once `vX.Y.0` is released; hotfixes are made on this branch; hotfixes are released immediately, often several times a day.
* `production` (project repositories only) - for use in production; no development will be done on this branch.

**Note**. When I say "develop on `v1` long-living branch", I don't mean it literally. In non-solo projects it is better to create a "feature branch" - short-living branch which you actually commit to and then merge it into long-living branch when ready.

## Version Tags

In library packages, every release is tagged with a `vX.Y.Z` version following [semantic versioning](https://semver.org/) rules. Accoring to semantic versioning, version number is incremented with each release as follows:

* `X` - a breaking change is released
* `Y` - a new feature is released (no breaking changes)
* `Z` - a bugfix is released (no new features or breaking changes)

For easier decision on version tag increment, start every commit message with `major:` to indicate a breaking change, `minor:` to indicate non-brekaing new features, and `fix:` for hotfixes. When releasing new version, review pending commit messages. If  there is at least one `major`, increase `X`; otherwise, if there is at least one `minor`, increase `Y`; otherwise, increase `Z`.

## Breaking Changes

What does "breaking change" mean? In general, it is a change in the public API which may break the calling code (or **user code**).

* For example, if user code may call some public method `$obj->render()` without parameters and you add some mandatory parameter to the signature of `render()` method, user code will break with error. Such change in public method signature is a breaking change. If, however, you add a parameter with a default value which makes the method to work the same way as before, it is not a breaking change.

* Another example of a breaking change is a change in project's REST API which may break the code using the REST API.

* Changing documented logic of a public method or REST API call is also a breaking change.

* Updating major dependency of a project or a library package to the next major version is also a breaking change. While it is not strictly "a change in the public API which may break the calling code", it is useful for version bookkeeping.

Don't treat changes in protected API as breaking changes even if you allow user code to extend your classes and use protected API. This way, user code is responsible for compatibility with your protected API.

## Project Directories

Keep you projects in a single directory (the `{home_dir}`), be it your user's home directory or some other directory.

For every `{project}`, maintain every long-living `{branch}` of the project in a separate directory `{branch}.{project}` (let's call it **environment**). For example,
for `osmdocs.com` project, I have several environments:

* `master.osmdocs.com`
* `v1.osmdocs.com`
* `production.osmdocs.com`

For Web projects, use environment name as a Web domain name.

## Dependency Setup

Configure `composer.json` in each environment differently. Continuing with the previous example, below are typical setups of every environment.

### `master` Environment

In `master` environment, two library packages the project uses are also on `master` branch. Here is how it is configured in `composer.json` files:

    // project's composer.json
    "require": {
        "osmphp/docs": "dev-master@dev",
        "osmphp/framework": "dev-master@dev"
    }

    // docs's composer.json
    "require": {
        "osmphp/framework": "dev-master"
    }

    // no require section in framework's composer.json

### `vX` Environments

In `v1` environment, both library packages are on `vX` branch of versions currently used in production (in the example below, on `v3` branch):

    // project's composer.json
    "require": {
        "osmphp/docs": "v3.x-dev",
        "osmphp/framework": "v3.x-dev"
    }

    // docs's composer.json
    "require": {
        "osmphp/framework": "v3.x-dev"
    }

    // no require section in framework's composer.json

### `production` Environment

In `production` environment, both library packages are on a version tag matching specified version constraint:

    // project's composer.json
    "require": {
        "osmphp/docs": "^3.0"
    }

    // docs's composer.json
    "require": {
        "osmphp/framework": "^3.0"
    }

    // no require section in framework's composer.json

## Workflow

* In `master` environment, do all the major development. When ready:
    * if release is major, create new `vX` environment from `master`
    * if release is minor, merge into existing `vX` environment
* In `vX` environment, develop hotfixes. When ready:
    * merge project file changes into `production`
    * create new version tags in modified library packages
    * merge `vX` into `v(X+1)`
    * merge the latest `vX` into `master`
* In `production` environment, pull `production` branch and run `composer update`

## Setup Variations

1. You can use `vX` environment in production and don't have `production` branch at all. In this case the production environment is on `vX` branches of the project and library packages, not on specific version tags. Less safety, but faster deployment.

2. In initial stage of the project development, you may omit `vX` branches and deploy `master` branches to the production environment.

3. If you don't need active previous versions of the project, you may have a single `v1` branch in project's Git repository and not create `v2` even in case incoming breaking changes.

## Common Pitfalls

### `composer.json` Merge Conflicts

Project and library package `composer.json` files in all the environments are different, so you can run into merge conflicts when releasing a hotfix.

Hotfixes are made on `vX` branch.

There is no merging when applying library package hotfixes to `production` - hence no problem.

However, all hotfixes should also be merged into `master`, and there MAY BE a problem if:

* you modify `composer.json` file in a hotfix AND
* `composer.json` file on `vX` and `master` branches are different.

    As in the example above, `docs` library package requires `dev-master` framework version on `master` branch and `v3.x-dev` framework version on `v3` branch.

In this case resolve merge conflict manually by keeping all changes made in the hotfix except framework dependency - it should remain `dev-master`.

In order to prevent Git to merge `composer.json` files automatically, add the following line to `.gitattributes` file in project and library package directories:

    /composer.json -merge

### `composer.lock` Merge Conflicts

Project's `composer.lock` is often different across project's `master`, `vX` and `production` branches. If you run `composer update` on a branch and then try merging it into another long-living branch, you may get a merge conflict.

To resolve such merge conflict, stay in merge conflict state, then run `composer update` and then commit.

In order to prevent Git to merge `composer.lock` files automatically, add the following line to `.gitattributes` file in project directory:

    /composer.lock -merge

### Packagist

Publish public library packages on [Packagist](https://packagist.org/) and configure webhooks to automatically update Packagist whenevr you push to Git repository.

Keep in mind that it takes for Packagist to update (1-2 minutes). Only then you can use `composer update`. To move faster, consider adding library package Git repositories to project's `composer.json` file.

For private library packages, always add their Git repositories to project's `composer.json` file.

Use the following command to add Git repository `{repo_url}` of `{vendor}/{package}` package:

    composer config repositories.{vendor}_{package} vcs {repo_url}

### Composer Failures

`composer update` may fail to replace branch constrained dependency (`dev-master`, `v1.x-dev`) with version constrained dependency (`^1.0`) or vice versa. In such cases, delete library package from `vendor` directory and then repeat `composer update`.
