---
layout: post
title: "Enhance Xcode templates using Git"
author:
  name: "Gaétan Zanella"
  link: "https://github.com/gaetanzanella"
tags: iOS macOS
---

Xcode provides a selection of default file and project templates. They act as a basis anytime you create a new file or a new project in Xcode.

<figure>
    <img src="/assets/enhance-xcode-templates-using-git/templates-window.png" height="300px" alt="Xcode template selection window"/>
    <figcaption>Xcode template selection window</figcaption>
</figure>

Since old times, we can create our own templates and integrate them easily in Xcode. `Paul Hudson` wrote a [great post](https://www.hackingwithswift.com/articles/158/how-to-create-a-custom-xcode-template-for-coordinators) about them.

Let see how we can use Git to power them up and create a distributed sample code codebase available right inside Xcode!

## Creating custom Xcode templates

An Xcode template is basically a folder suffixed with the `.xctemplate` path extension containing some source files and some metadata. Here is an example of a [CoreData template](https://github.com/faberNovel/CodeSnippet_iOS/tree/master/XCTemplate/Data/ADCoreDataStack.xctemplate).

To populate its template selection window Xcode looks for templates in two distinctive folders on the hard drive:

- The Xcode built-in folder: `/Applications/Xcode.app/.../Templates`
- The current user folder: `~/Library/Developer/Xcode/Templates/`

Adding a custom template in Xcode is as simple as copying and pasting a correctly structured `xctemplate` folder into `~/Library/Developer/Xcode/Templates/`.

<figure>
    <img src="/assets/enhance-xcode-templates-using-git/custom-templates-window.png" height="300px" alt="CoreData template available in Xcode"/>
    <figcaption>Example of a CoreData template available in Xcode</figcaption>
</figure>

`Paul Hudson`'s [post](https://www.hackingwithswift.com/articles/158/how-to-create-a-custom-xcode-template-for-coordinators) explains in details the Xcode template structure. Consider reading it if you want to master the template creation!

## Installing Xcode templates from git

In his post, Paul does not offer advice on how to manage Xcode templates. If we store our Xcode templates inside an Xcode configuration folder, we are at an increased risk of losing them and we would not be able to share them accross multiple machines. Storing our templates inside a git repository sounds like the best way to go. 

But manually downloading a git repository into the Xcode configuration folder anytime we need to update our templates sounds fastidious though. How can we link templates from a repository to Xcode?

[xcresource-cli](https://github.com/faberNovel/xcresource-cli) can do just that. It is a small tool aimed at managing Xcode templates. In particular, it can download templates from a specified repository URL and integrate them in Xcode:

{% highlight ssh %}
> xcresource template install --url https://github.com/faberNovel/CodeSnippet_iOS --namespace FABERNOVEL --pointer main
{% endhighlight %}

<figure>
    <img src="/assets/enhance-xcode-templates-using-git/fabernovel-templates-window.png" height="300px" alt="xcresource can install templates from a git repository"/>
    <figcaption>xcresource can install templates from a git repository</figcaption>
</figure>

During the installation, we can specify a `namespace` parameter. It acts as an installation folder and allows us to install templates from different sources:

{% highlight ssh %}
// install the templates from the main branch of the repo
> xcresource template install --url https://github.com/faberNovel/CodeSnippet_iOS --namespace MAIN --pointer main

// install the templates from the develop branch of the repo
> xcresource template install --url https://github.com/faberNovel/CodeSnippet_iOS --namespace DEVELOP --pointer develop

// only remove the folder containing the templates from the develop branch of the repo
> xcresource template remove --namespace DEVELOP
{% endhighlight %}

## Using Xcode templates as a distributed sample code database

Coupling Xcode templates with git can be very powerful. On one hand, Git allows us to distribute our templates and make contributing to them easy. On the other hand, thanks to the flexibility offered by Xcode, our templates can be displayed and selected right inside Xcode.

<figure>
    <img src="/assets/enhance-xcode-templates-using-git/distributed-sample-code-database.png" height="280px" alt="xcresource can install templates from a git repository"/>
</figure>

At Fabernovel, we use such sample code database. All our sample codes are stored in the [code snippets repository](https://github.com/faberNovel/CodeSnippet_iOS) which acts as a sample code reference repository. Anytime a sample code wouldn't fit inside a library because it is too small, too variable or simple boilerplate code, we store it inside this repository. It mainly contains basic UI elements, boilerplate code, helper extensions, algorithms etc. Because all the sample codes are written as [Xcode templates](https://github.com/faberNovel/CodeSnippet_iOS/tree/master/XCTemplate), they are all available in Xcode anytime we create a new file thanks to `xcresource-cli`.

The code snippet repository perfectly complements the tools we already use in our Fabernovel developer day-to-day life to be more productive and avoid boilerplate:

- A project templater we use anytime we start a new iOS project
- [ADUtils](https://github.com/faberNovel/ADUtils) a set of helpers, shortcuts or other tools providing simplified interactions with UIKit and more generally with Swift.
- [ccios](https://github.com/felginep/ccios) an Xcode file generator for Clean Code architecture

## What's next?

Try to create your own Xcode template repository and display them in Xcode of course!

You can read [Hudson's post](https://www.hackingwithswift.com/articles/158/how-to-create-a-custom-xcode-template-for-coordinators) and [xcresource-cli's readme](https://github.com/faberNovel/xcresource-cli) to get a better overview of what you can do with Xcode templates and Git.

Discover how you can use `xcresource` to [enhance your Xcode codesnippets](https://fabernovel.github.io/2021-07-22/enhance-xcode-snippets-using-git) ;)
