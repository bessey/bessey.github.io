---
layout: post
title: "Github Releases are no match for CHANGELOG.md"
date: 2024-06-18 13:01
comments: true
categories: changelog github ux
---

I have noticed a trend recently of library authors on Github forgoing maintaining a [CHANGELOG.md](https://keepachangelog.com/en/1.1.0/) file, in favour of exclusively using Github Releases.

![Example of Github repository changelog.md stating that "Release notes are now stored in Github Releases"](/images/changelog/move-to-releases.png)

For a library author, the choice is understandable. It takes an effort to maintain a useful changelog that strikes the right balance of capturing significant changes to library without ending up as a list of every commit. But there's a pretty big problem with this... Github Releases today are a much worse solution to the problems changelogs solved for a library's **users**.

<!-- more -->

Please note, throughout this article I will draw on examples from many libraries. I do not intend to target any specific authors, this practice is widespread.

First off, let me clarify what I use changelogs for, and what I expect of them. By far the most common time I use a changelog is when upgrading a library, to understand what significant changes have occurred between the version I'm on and the version I'm upgrading to. E.g. what changed between `v3.22.0` to `v3.23.8`? Pretty simple stuff, but let's see how the experiences compare between the two approaches as a user.

In both cases, I start by navigating to the Github repo of the library I'm upgrading. From there we diverge...

### Github Releases

We navigate to the Releases page with one click, where we can see the 10 most recent releases. Most of them are patches, so we only get one minor release on the whole page.

![](/images/changelog/releases-index.png)

This isn't the version we're upgrading from though, obviously. It's ok though, there's a search bar, so we can just search for my exact version! Except...

![](/images/changelog/releases-bad-search.png)

Nope, the Releases search bar behaviour is... *unintuitive*. Passing the exact Release tag string doesn't guarantee it comes up first, or even on the first page. Adding quotation marks doesn't either. It [turns out](https://docs.github.com/en/repositories/releasing-projects-on-github/searching-a-repositorys-releases#search-syntax-for-searching-releases-in-a-repository) the magic incantation is `tag:blah`:

![](/images/changelog/releases-search-better.png)

Great! We have the version we're running now. Now we just need to page forwards through the newer versions. Except, that is not possible from this view:

![](/images/changelog/releases-search-no-pager.png)

So I guess we'll have to return to the index page backwards from the latest version all the way to the version we are currently on after all. In batches of 10. Fun.

![](/images/changelog/releases-pager.png)

Well this is awful. Oh, at least there's a `&page=X` query param. We can use that to skip back and forth through pages honing in on the one with our release version in it. We go to `page=3` and the version is too new. On `page=8` the version is too old. How about `page=5`? Too new again... **Somehow we have gone from upgrading a library to executing a human powered binary search**.

Finally though we do find the version we are on, and can now start paging forwards through the newer versions, checking for breaking and other significant ([to you](https://xkcd.com/1172/)) changes. Even now though we run into an annoying behaviour we need to be aware of. Since releases are ordered chronologically by **publish** date there may be releases "between" versions in the list that do not apply to us because they are not present in the version we are running, because they are from another branch:

![](/images/changelog/releases-out-of-order.png)

Mercifully it is possible to suppress pre-releases with `prerelease:false` in the search UI. However, this doesn't solve the issue where older major versions are maintained alongside newer versions.

All in all, quite the ordeal to answer a simple question. So how does this modern clean solution compare to the ancient archaic text file approach?

### CHANGELOG.md

We navigate to the CHANGELOG.md with one click, `Ctrl+F` for our version, type its name into the browser find in page UI, and we are instantly scrolled down to it. If you hate typing, we also have a list of jump links to every version.

![](/images/changelog/changelog-ctrl-f.png)

It's not uncommon to have the entire repository's history in this single page. [React](https://github.com/facebook/react/blob/4ddff7355f696ec693c5ce2bda4e7707020c3510/CHANGELOG.md#030-may-29-2013) for example has **11 years of history** on this one page.

And to see changes between the upgraded version and ours? Scroll up. That's it 😄. By virtue of being branch specific, changelogs sidestep the aforementioned pre-release problem, because the repository's trunk branch changelog only contains changes that have landed on the trunk. If you _are_ a pre-release tester, the release branch's changelog has what you need.

----

I think the difference in UX of these two workflows speaks for themselves. I have two proposals going forwards...

If you are a **Github employee**, please, improve this experience! We have long had a UI for "compare git diff between commits". Now we need a UI for "compare releases diff between (semantic) release tags".

If you are a **repo maintainer** that has migrated entirely to Github Releases, please consider maintaining a CHANGELOG.md as well. There are excellent resources to help you here [[1](https://keepachangelog.com/en/1.1.0/), [2](https://changelog.md/)].
