---
layout: post
current: post
cover: assets/images/posts/gradle-git.png
title: Git repos as Gradle dependencies
date: 2020-05-12 07:00:00
tags: Gradle
class: post-template
author: alexvasilkov
---

It is not an uncommon desire to split your codebase into reusable libraries (git repositories).
Let’s review what options we have if we want to include external code into a Gradle project.

## Git submodules

An obvious choice can be git submodules. Creating a git submodule is easy:
{% highlight sh %}
git submodule add {url} {name}
{% endhighlight %}

And then you need to manually download submodules with
{% highlight sh %}
git submodule update --init --recursive
{% endhighlight %}

Looks simple, but using git submodules can quickly become painful.
GitHub [does not recommend][GitHub-Submodules] to use it in most cases, because:

* Git doesn’t download submodule contents by default.
* Difficult to contribute changes back to the submodule repository.
* Collaborators won’t automatically see updates to submodules.

If you have a simple use case and want to include an external repository that will change rarely
then this option can be good enough.

##  Binary artifacts

The most popular option is to publish shared code as binary (`.jar` or `.aar`) artifacts
in an external Maven repository.

Open source libs can be distributed with one of a few free public Maven repositories like
[Sonatype][Sonatype], [Bintray][Bintray], etc. But if you want to distribute privately within a
company then you’ll need to maintain your own private Maven repository
(e.g. [Artifactory][Artifactory]), pay for external services (e.g. [Bintray][Bintray-Private]),
or try to [use git repo][SO-Git_as_mvn] as a Maven repository.

If you store your code on GitHub then you can use [GitHub Packages][GitHub-Packages],
it is also available for private repositories, the main downside of this service is a storage limit.

Once published it becomes very easy to use the library by adding it as a regular Gradle dependency.
But this approach has a few important limitations:

* Requires private Maven repository for non-public libraries.
* Requires non-trivial build script configuration.
* Requires extra efforts to publish new versions. Can be a problem for frequently changing projects.
* Testing library code in a host project requires publishing a new snapshot version of the library,
and repeating it until all issues are resolved.
* Android flavors and build modes are not supported.
Separate artifacts should be published for different lib’s variants.

Overall this is a great option for public libraries, for big companies, or when no frequent changes
are expected. In other cases it may become frustrating to make changes to the libraries and maintain
the necessary infrastructure.

## JitPack

[JitPack][JitPack] is a nice service that builds git repos on their servers on your behalf,
it supports lots of git hostings (GitHub, Bitbucket, Gitlab), supports
[private repositories][JitPack-Private] and even supports Android flavors. The usage is simple:

{% highlight groovy %}
repositories {
    maven { url 'https://jitpack.io' }
}
dependencies {
    implementation 'com.github.User:Repo:Tag'
}
{% endhighlight %}

Overall it’s an interesting combo of a regular (“manual”) Maven repository and CI server.
It is indeed much easier to set up a library and release new versions with JitPack. Though it still
has some of drawbacks of a regular binary distribution:

* Requires paid monthly subscription for non-public libraries.
* Testing library code in a host project requires pushing changes to remote repo and waiting
for JitPack to build it.
* Less control of the distribution process. E.g. no checks and tests (unless using JitPack CI),
end users may use any commits / branches as if they were real versions.

## Gradle source dependencies

Source dependencies feature was [introduced][Gradle-Source_dep] back in Gradle 4.x,
it works by including a regular dependency:

{% highlight groovy %}
dependencies {
    implementation 'org.gradle.cpp-samples:utilities:1.0'
}
{% endhighlight %}

and specifying how to get it from external git repo, in `settings.gradle`:

{% highlight groovy %}
sourceControl {
    gitRepository('https://github.com/gradle/native-samples-cpp-library.git') {
        producesModule('org.gradle.cpp-samples:utilities')
    }
}
{% endhighlight %}

Gradle will download the sources from the (public) git repository and will try to build a binary
artifact out of it. But this option has rather strict limitations:

* Only public repositories are supported at the moment (see [this bug report][Gradle-Auth_bug]).
* The project in git repo should be properly configured so that Gradle knows how to build and identify its artifact.
* Plus most of the same issues from the binary distribution option, as stated above.

Overall this option does not look promising, it has lots of limitations and a fairly complicated setup.

## Gradle plugin

Another alternative is to use a [Gradle plugin][GradlePlugin] (written by the author of this
article). It helps to fetch and attach external git repositories as dependencies.
For the last 5 years we successfully used the first
version of this plugin in our company, and now I’m glad to introduce the second version,
with a cleaner DSL and some minor fixes. Let’s check the usage.

In `settings.gradle` file, declare the plugin:

{% highlight groovy %}
plugins {
    id 'com.alexvasilkov.git-dependencies' version '2.0.1'
}
{% endhighlight %}

<p class="code_comment">
Note that for Gradle prio 6.x a more verbose declaration is needed, see plugin docs.
</p>

In module’s `build.gradle` file, declare a git repository as dependency:

{% highlight groovy %}
git {
    implementation 'https://example.com/repository.git', {
        tag 'v1.2.3'
    }
}
{% endhighlight %}

Now the plugin will take care of downloading (or updating) specified git repo into `libs/{git name}`
directory and attaching it as a [project dependency][Gradle-Project_dep].

Key features:

* Both https and ssh authorization are supported.
* Dependencies are resolved recursively, i.e. your git repository can define other git repositories
as dependencies.
* Versions conflict detection. If the same repository is declared with different commit, tag or
branch then the build will fail.
* Attached repositories can be safely edited and tested directly in the host project.
* Seamless Android flavors support, since git repos are treated as regular modules.

The plugin will resolve and download git repos before Gradle projects (modules) are evaluated,
so one of the cool features is that you can move all your build scripts into a separate repo as well
and then reuse them by declaring in `settings.gradle` like this:

{% highlight groovy %}
git {
    fetch 'https://example.com/build_scripts_repo.git', {
        dir "$rootDir/gradle/scripts"
        tag 'v1.2.3'
    }
}
{% endhighlight %}

Now your shared build scripts are available in all the projects with:

{% highlight groovy %}
apply from: "$rootDir/gradle/scripts/{your_script}.gradle"
{% endhighlight %}



In general this option is similar to git submodules but without most of its issues.
There are still some issues worth mentioning:

* Requires downloading and compiling source code, in contrast to pre-compiled binary artifacts.
* No Kotlin DSL support yet.
* No official support / maintenance from big companies to rely on.

## Conclusion

It is always a good idea to separate and reuse repeatedly used code into a dedicated library
project. For public libraries binary artifacts distribution is the most popular
and recommended option, while private libraries distribution can be trickier. If you wish to avoid
building and distributing binary artifacts within your company then the Gradle plugin presented in
this article can be a good alternative. As a side effect it also provides a mechanism to reuse your
build scripts logic among several projects.


[GradlePlugin]: https://github.com/alexvasilkov/GradleGitDependenciesPlugin
[GitHub-Submodules]: https://github.blog/2016-02-01-working-with-submodules/
[Sonatype]: https://www.sonatype.com/nexus-repository-oss
[Bintray]: https://bintray.com/bintray/jcenter
[Artifactory]: https://www.jfrog.com/confluence/display/RTF6X/Installing+Artifactory
[GitHub-Packages]: https://github.com/features/packages
[Bintray-Private]: https://jfrog.com/article/private-repositories/
[SO-Git_as_mvn]: https://stackoverflow.com/questions/14013644/hosting-a-maven-repository-on-github
[JitPack]: https://jitpack.io/
[JitPack-Private]: https://jitpack.io/docs/PRIVATE/
[Gradle-Source_dep]: https://blog.gradle.org/introducing-source-dependencies
[Gradle-Auth_bug]: https://github.com/gradle/gradle/issues/8245
[Gradle-Project_dep]: https://docs.gradle.org/current/userguide/declaring_dependencies.html#sub:project_dependencies
