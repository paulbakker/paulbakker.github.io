---
layout: post
title:  "Amdatu Bootstrap r8"
date:   2015-06-26 15:56
categories: Amdatu
---

We've just released a new version of Amdatu Bootstrap, packed with fixes and new features. 

Support for Apache Felix Dependency Manager 4
==
The release of Apache Felix Dependency Manager 4 is important for many projects, specially for projects using Amdatu. Bootstrap now uses version 4 by default when installing it to a project and in all templates. Optionally version 3 can still be used in the install and run commands.

To facilitate upgrading a project to DM 4 we also added a plugin to help with this. "dm-update" will update the annotation processor in the workspace configuration and all project bnd files. You can use the existing "versions" plugin to upgrade build and run configurations.

The name of the DM plugin in Bootstrap is also renamed to "dm" instead of "dependencymanager". Because this is an often used plugin, the shorter name improves usability, and the "dependencymanager" name comflicted with other plugins in command completion. So now you simply use:

{% highlight bash %}
dm-install
dm-run
{% endhighlight %}

Help feature
==
The number of commands supported by Bootstrap has exploded in the past months. This is great because it means more features. It also introduced a new problem; how do you figure out which command to use as a new user!?
To solve this we introduced a "help" button in the UI, which shows a (dynamic) listing of all plugins/commands with their description. So make sure to add a description when authoring plugins!

Improved WebUI plugin
==
The WebUI has been improved a lot in this release. This plugin is used to generate projects with a basic structure to create AngularJS/TypeScript projects. 

{% highlight bash %}
webui-install
{% endhighlight %}

This creates an "app" directroy in the project, containing the AngularJS/TypeScript project. Because this is a TypeScript project, you need to compile the project before running in a browser. A Grunt build is added to the project as well, that takes care of TypeScript compiling, minifying and concatination. This means the project is fully prepared for production deployments. The compiled and minified resources are copied to the "dist" folder which is included in the OSGi bundle.The Grunt build is automatically invoked by Gradle, so there are no extra build steps required.

For development we advise using an editor optimised for web development, such as WebStorm. You can run the UI directly from a local web server (as included in tools like WebStorm), so that you don't have to re-build the OSGi bundle during development. Using the following command will watch the project and compile the TypeScript code on changes.

{% highlight bash %}
grunt dev
{% endhighlight %}

If Grunt and the required plugins are not installed yet you can simply run the following command in the ui project to do so.

{% highlight bash %}
npm install
{% endhighlight %}

If you are unfamiliar with Grunt and TypeScript this might seem like a lot of extra steps. This workflow is optimised for large scale web development, which needs different tools than OSGi development. We highly encourage to get familiar with these tools to get the best development experience if you're into web development.

Baseline repos and workspace materialisation
==

We have some new interesting plugins to create indexed repositories from a Git tag of your project, or the bundles listed in a bnd(run) file. One of the use cases is creating a local repository containing the bundles of a previous version of the project (using a Git tag) that can be used to baseline against. Online/shared repositories are very fragile when it comes to baseling and using different compiler settings etc. By building a baseline repository locally these problems are greatly reduced. Such a repository can be build using the command below. Note that the project is checked out from Git, built using Gradle and finally indexed as a repository.

{% highlight bash %}
repositories-materializeWorkspace
{% endhighlight %}

An example of this process can be seen in the video below.

<iframe src="https://player.vimeo.com/video/117951296" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<br><br>

You can also generate repositories directly from bnd(run) files as seen in yet another video.

<iframe src="https://player.vimeo.com/video/116064362" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<br><br>

Get it while it's hot!
==

[Download](https://bitbucket.org/amdatu/amdatu-bootstrap/downloads/bootstrap-r8-bin.zip) the latest release and a get going! If you have any issues or ideas for new features, let us know either on the [mailing list](http://amdatu.org/mailinglist.html) or [Jira](https://amdatu.atlassian.net/projects/AMDATUBOOT).