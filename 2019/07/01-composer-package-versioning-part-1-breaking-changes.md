# Composer Package Versioning - Part 1 - Breaking Changes #

This article series is about software versioning. This is the first, introductory article in the series. 

Contents:

{{ toc }}

> **Note**. Examples in this series are tailored for PHP, [Composer](https://getcomposer.org/) and [Git](https://git-scm.com/), though same principles apply to other programming languages and version control systems as well.  

## To Update Or Not To Update ##

Let's say you develop a PHP project using PHP library `some/library`, developed by someone else:

	project --> some/library  

As `some/library` evolves (bugs are getting fixed, new features are introduced), you face a dilemma: installing `some/library` updates may benefit the project, but it may also break the project. 

Versioning helps you with this decision:

1. You can install not only the latest version, but **any** version ever released.
2. Developer of `some/library` may specify, which versions can be installed without much thought and which should be installed with caution.
3. You may automate installing harmless updates while keeping potentially dangerous updates on hold and install them manually.

## Communicating Breaking Changes ##

Updates which may break the project are called **breaking changes**.

As it is the library developer who have to tell the project developer if library version contains breaking changes or not, let's see the same situation from the library developer point of view. 

[Semantic Versioning](https://semver.org/) offers you as the library developer a clear set of rules for communicating breaking changes. Summary:

* Assign every new version of the library `X.Y.Z` version number where `X` - is major version, `Y` is minor version and `Z` is patch number.
* When you release new package version (let's say that current version is `1.0.0`) you communicate to users of your library using one of 3 messages:
	* increase patch number (`1.0.1`): new bug fix is released, won't break anything, just update your projects and don't worry.
	* increase minor version (`1.1.0`): we introduced new features, but it should not break anything (but may in some rare case), update and shallow test.
	* increase major version (`2.0.0`): we introduced new ground breaking features, test thoroughly after update.

## What's Next ##

[In the next part](../08/composer-package-versioning-part-2-git-branches-and-tags.html), I will review how to keep up with semantic versioning using Git.

[Discuss on HN](https://news.ycombinator.com/item?id=20527677)