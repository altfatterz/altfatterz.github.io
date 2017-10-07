---
layout: post
title: Install a specific version of a formula with homebrew
tags: [MacHomebrew]
---

Installing software packages on Mac is very easy with [homebrew](https://brew.sh/). You typically get the latest version, however often in production you do not have the latest version of a software package. Another use case is when you upgrade to the latest and you find out there is bug which blocks you doing something. In this case you would like to downgrade to the previous version until the bug is fixed.
In both cases you need to install a specific version of a software package with homebrew on your Mac, which tends to be not that trivial. There is a lot of discussion about this on stackoverflow but some of them are outdated based on `brew versions` which is not available anymore.   

Let's look at an example installing my favourite relational data store: PostgreSQL. With `brew search postgresql` at the time of writing we get the following formulas: `postgresql`, `postgresql@9.5` and `postgresql@9.4` which correspond to the [postgresql.rb](https://github.com/Homebrew/homebrew-core/blob/master/Formula/postgresql.rb), [postgresql@9.5](https://github.com/Homebrew/homebrew-core/blob/master/Formula/postgresql%409.5.rb) and [postgresql@9.4](https://github.com/Homebrew/homebrew-core/blob/master/Formula/postgresql%409.4.rb) formula definitions.
If you `brew install` these three formulas then you will get at the time of writing versions 9.6.5, 9.5.9 and 9.4.14.

Now let's imagine in production you are running version 9.6.4 and you would like to test against this version on you Mac.  
In order to install the 9.6.4 version we need to downgrade the homebrew installation to the commit which contains the upgrade of PostgreSQL to 9.6.4  
 
```bash
cd "$(brew --repo homebrew/core)"
git log Formula/postgresql.rb
```

The output is something like:

```bash
...

commit c48e1cf03592fd450f2f0a6db883e8c12f450c13
Author: BrewTestBot <brew-test-bot@googlegroups.com>
Date:   Thu Aug 10 23:46:21 2017 +0000

    postgresql: update 9.6.4 bottle.

commit e29fb923a796985e78612a8ca7fe4a62037a19a6
Author: Dominyk Tiller <dominyktiller@gmail.com>
Date:   Fri Aug 11 00:06:39 2017 +0100

    postgresql 9.6.4

    Fixes multiple CVEs. If you have an existing database that you don't
    intend to blow away & regenerate you may wish to read:
    https://www.postgresql.org/about/news/1772/
...    
   
```   
 
You need to set the `HOMEBREW_NO_AUTO_UPDATE` to force homebrew not to auto-update before running `brew install`.
You need also to unlink the postgresql before the `brew install` otherwise the command will fail.

```
git checkout -b postgresql-9.6.4 c48e1cf03592fd450f2f0a6db883e8c12f450c13
brew unlink postgresql
HOMEBREW_NO_AUTO_UPDATE=1 brew install postgresql
```

After the installation the new version is automatically linked.

```bash
postgres --version
9.6.4
``` 

Let's cleanup by reverting the downgrade of our homebrew installation.

```bash
git checkout master
git branch -d postgresql-9.6.4 
```

You can check the different installed versions with `brew list <formula> --versions` command.

```bash
brew list postgresql --versions
postgresql 9.6.4 9.6.5
``` 

You can find more information about a formula with `brew info` where the `*` character specifies which version of the formula is currently linked. 

```bash
brew info postgresql
...
/usr/local/Cellar/postgresql/9.6.4 (3,264 files, 36.6MB) *
  Poured from bottle on 2017-10-07 at 15:44:57
/usr/local/Cellar/postgresql/9.6.5 (3,269 files, 36.7MB)
  Poured from bottle on 2017-09-25 at 13:47:06
From: https://github.com/Homebrew/homebrew-core/blob/master/Formula/postgresql.rb
...
```

Now you can easily switch between the versions with `brew switch`

```bash
brew switch postgresql 9.6.5

Cleaning /usr/local/Cellar/postgresql/9.6.4
Cleaning /usr/local/Cellar/postgresql/9.6.5
375 links created for /usr/local/Cellar/postgresql/9.6.5

postgres --version
9.6.5
```

Installing a very old version of a specific software package might not work since you basically revert your homebrew installation and the commands might change. 
But for installing previous minor updates of a formula should not be a problem.