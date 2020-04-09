---
layout: post
title: Semantic versioning with jgitver
tags: [jgitver, semanticversioning, github]
---

[`jgitver`](https://jgitver.github.io/) is a very useful tool to automatically compute the version of your project leveraging the git history.
It does not pollute the project's git history like the [maven release plugin](http://maven.apache.org/maven-release/maven-release-plugin/) 
 
`jgitver` has support for both Maven and Gradle. In this blog post we are going to use Gradle. 

```bash
plugins {
    id 'fr.brouillard.oss.gradle.jgitver' version '0.9.1'
}
```

`jgitver` comes with many good defaults which follow best practices and conventions which can be customised if needed.

It automatically registers a task `version`. When executing it on a non git repository then we get the following result:

```bash
$ ./gradlew version

Version: 0.0.0-NOT_A_GIT_REPOSITORY
```

And indeed after `./gradlew build` we have an `<artifact-id>.0.0.0-NOT_A_GIT_REPOSITORY.jar` in the `build/libs` folder. 

Let's make it a git repository with `git init` and verify that `./gradlew version` returns `0.0.0-EMPTY_GIT_REPOSITORY`.

Next, adding our fist commit we have the version `0.0.0-0`. When no tags can be found in the commit history, `jgitver` defaults to a virtual lightweight git tag `0.0.0` on the first commit. Any additional commits will increment the last number, `0.0.0-1`, `0.0.0-2`, etc.

The last number signals how far (how many commits) are we from the last lightweight git tag. (in the above case the `0.0.0` virtual lightweight tag)

### Lightweight git tag

Let's create a lightweight git tag and check the project version:

```bash
$ git tag 0.0.1 
$ ./gradlew version
Version: 0.0.1-0
```

### Annotated git tag

Let's create an annotated git tag and check the project version:

```bash
$ git tag -a 0.1.0 -m "first stable version"
$ ./gradlew version
Version: 0.1.0
```

As we see with annotated git tag the calculated version is the tag name. Any further commits create versions like `0.1.1-1`, `0.1.1-2`, etc. Beside incrementing the last number, it also moved the first part of the version from `0.1.0` (based on our exiting annotated git tag) to `0.1.1`.  
And again if we create another annotated git tag, be it `0.1.1` (minor fixes) or `0.2.0` (new features but still backwards compatible) or `1.0.0` (incompatible API change) then the calculated version will be always equal to the annotated git tag.

### Branching

Let's say we are on the `0.2.0` annotated git tag from where we create our `awesome-feature` branch then the calculated project version will be the following:

```bash
$ git checkout -b awesome-feature
$ ./gradle version
$ Version: 0.2.0-awesome_feature
```  

Any further commits on the `awesome-feature` branch will produce the following versions: `0.2.1-1-awesome_feature`, `0.2.1-2-awesome_feature`, etc.

### Expose version

It comes handy when the service exposes its version number. Using Spring Boot this is very handy, we just need to include

```bash
springBoot {
    buildInfo()
}
```

The above configuration makes sure that build information is exposed via the `/info` actuator endpoint.

```bash
$ http :8080/actuator/info

{
    "app": {
        "description": "jgitver calculates a project semver compatible version from a git repository",
        "name": "jgitver-demo"
    },
    "build": {
        "artifact": "jgitver-demo",
        "group": "com.example",
        "name": "jgitver-demo",
        "time": "2020-04-09T20:50:58.276Z",
        "version": "0.0.2-1"
    },
    "git": {
        "branch": "master",
        "commit": {
            "id": "de5248a",
            "time": "2020-03-07T21:46:14Z"
        }
    }
}
``` 

In the above example git information is also exposed using the [gradle-git-properties](https://plugins.gradle.org/plugin/com.gorylenko.gradle-git-properties) plugin.

### Docker

Since nowadays the build artifact is a docker image we want to make sure the image is tagged with calculated project version.
Here we are using [Jib](https://github.com/GoogleContainerTools/jib) to dockerize our Spring Boot application. If you are new to Jib have a look to my previous blog post [https://zoltanaltfatter.com/2019/08/16/dockerizing-spring-boot-apps-with-jib/](https://zoltanaltfatter.com/2019/08/16/dockerizing-spring-boot-apps-with-jib/))

To build to a Docker daemon, we can use 

```bash
$ ./gradlew jibDockerBuild 
```    

And indeed the docker image was created with the tag equal to the project's version calculated by `jgitver`.

```bash
$ docker images | grep jgitver-demo
jgitver-demo              0.0.2-1             a5418196da55        2 minutes ago       143MB
```

### Pipeline

But we don't create docker images locally, we do that with a build pipeline. We are going to use [GitHub Actions](https://github.com/features/actions)  
to build the artifact and publish it to [GitHub Packages](https://github.com/features/packages)

The workflow is [triggered](https://help.github.com/en/actions/reference/events-that-trigger-workflows) on `push` event.

```yaml
name: Build

on:
  push:

jobs:
  build:
    runs-on: ubuntu-latest
```
 
We use standard actions for [checking out](https://github.com/actions/checkout) the repository and [setting up java version](https://github.com/actions/setup-java).
The checkout action only fetches a single commit, but in order to `jgitver` to calculate the project version from git history we need all the tags.

```yaml
    steps:
      - name: Checkout latest code
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Fetch all tags
        run: git fetch --depth=1 origin +refs/tags/*:refs/tags/*

      - name: Set up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8

```

Then after restoring the Gradle build cache we build the project.

```yaml
      - name: Setup build cache
        uses: actions/cache@v1
        with:
          path: ~/.gradle/caches
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle') }}
          restore-keys: |
            ${{ runner.os }}-gradle-

      - name: Build with Gradle
        run: ./gradlew build
```

In the next step we build the docker image and push it to `GitHub Packages` registry.

```yaml
      - name: Compute version
        id: compute_version
        run: |
          echo "::set-output name=version::$(./gradlew version | grep Version | awk '{ print $2 }')"
      - name: Build and Upload Docker image
        run: |
          echo ${{ secrets.GITHUB_TOKEN }}
          ./gradlew jib \
            -Djib.to.image=docker.pkg.github.com/altfatterz/jgitver-demo/jgitver-demo:${{ steps.compute_version.outputs.version }} \
            -Djib.to.auth.username=altfatterz \
            -Djib.to.auth.password=${{ secrets.GITHUB_TOKEN }}
``` 

As you can see our pipeline is pretty fast, it took only 41 seconds.

![graphql-actions](/images/2020-04-10/github-actions.png)

And the produced docker image is in the GitHub Package Registry:

![github-packages](/images/2020-04-10/github-packages.png)

If we click on the version we can see the details how to pull the image and also some download statistics and previous versions.  
You cannot remove packages from GitHub Package Registry.

![github-packages-details](/images/2020-04-10/github-packages-details.png)

The example source code you can nfind here [https://github.com/altfatterz/jgitver-demo](https://github.com/altfatterz/jgitver-demo)   