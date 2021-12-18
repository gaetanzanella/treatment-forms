---
layout: post
title: "Enhance Xcode snippets using Git"
author:
  name: "Gaétan Zanella"
  link: "https://github.com/gaetanzanella"
tags: iOS macOS
---

In a previous [post](https://fabernovel.github.io/2021-06-22/enhance-xcode-templates-using-git) we described how to combine [xcresource](https://github.com/faberNovel/xcresource-cli) and [Xcode templates](https://www.hackingwithswift.com/articles/158/how-to-create-a-custom-xcode-template-for-coordinators) to create a distributed sample code database available right inside Xcode. 

This time, we will see how to use [xcresource](https://github.com/faberNovel/xcresource-cli) to share and back Xcode codesnippets up!

## Backing codesnippets up in a git repository

[Xcode snippets](https://sarunw.com/posts/how-to-create-code-snippets-in-xcode/#what-is-code-snippet%3F) are reusable blocks of code available in Xcode. You can browse them and create your own snippets in the [Xcode snippets library](https://sarunw.com/posts/how-to-create-code-snippets-in-xcode/#code-snippets-library).

Anytime you create a codesnippet, Xcode stores it in a local folder (`~/Library/Developer/Xcode/UserData/CodeSnippets`). You can use `xcresource snippet open` to browse them.

<figure>
    <img src="/assets/enhance-xcode-snippets-using-git/local-snippets.png" height="300px" alt="xcresourcecodesnippets storage directory"/>
    <figcaption>Xcode snippets directory</figcaption>
</figure>

Thanks to [xcresource](https://github.com/faberNovel/xcresource-cli), you can move your snippets to a [git repository](https://github.com/faberNovel/CodeSnippet_iOS/tree/master/XCSnippet) to back them up and install them again later locally. It is considered as a good pratice to rename them with explicit names.

Consider deleting your local snippets once copied, before installation, to avoid duplications:

{% highlight ssh %}
> xcresource snippet remove
> xcresource snippet install --url url_to_git_repo --pointer branch_name
{% endhighlight %}

## Installing snippets from different sources

When installing snippets, you can specify a `namespace` parameter:

{% highlight ssh %}
> xcresource snippet install --url url_to_git_repo --pointer branch_name --namespace my_namespace
{% endhighlight %}

A namespace acts as an installation folder. The snippets will be installed inside it. If the namespace already exists, it is replaced. Thus, you can use namespace to install snippets from different sources:

{% highlight ssh %}
> xcresource snippet install --url url_to_my_personal_git_repo --namespace personal_snippets

> xcresource snippet install --url url_to_my_company_git_repo --namespace company_snippets

> xcresource snippet list
# personal_snippets
- personal snippet 1
- personal snippet 2
# company_snippets
- company snippet 1
- company snippet 2
{% endhighlight %}

## A word on the snippet namespace implementation

When installing snippets, `xcresource` stores the namespace you specified in the summary of your snippets:

<figure>
    <img src="/assets/enhance-xcode-snippets-using-git/namespace.png" height="300px" alt="The snippet namespace is stored in its summary"/>
    <figcaption>The snippet namespace is stored in its summary</figcaption>
</figure>

Be careful to not override it when editing your snippets using Xcode.

## What's next?

Try to create your own Xcode snippets repository and display them in Xcode of course!

Discover how you can use [xcresource](https://github.com/faberNovel/xcresource-cli) to [enhance your Xcode templates](https://fabernovel.github.io/2021-06-22/enhance-xcode-templates-using-git) ;)
