---
layout: post
title: "Github Releases are no match for CHANGELOG.md"
date: 2024-06-18 13:01
comments: true
categories: changelog github ux
---

I have noticed a trend recently of library authors abandoning maintaining a [CHANGELOG.md](https://keepachangelog.com/en/1.1.0/) file, in favour of exclusively using Github Releases.

![Example of Github repository changelog.md stating that "Release notes are now stored in Github Releases"](/images/changelog/move-to-releases.png)

For a library author, the choice is understandable. A Github Release message contains much the same information as a changelog, so why duplicate effort and risk having no source of truth by maintaining both? But there's a pretty big problem with this: Github Releases today suck UX wise for your **users**.

_**TLDR?** Get the best of both worlds by generating your changelog from your Releases with [**rhysd/changelog-from-release**](https://github.com/rhysd/changelog-from-release)!_

<!-- more -->

First off, let me clarify what I use changelogs for, and what I expect of them. By far the most common use case I have for a changelog is when upgrading a library, to understand what significant ([to me](https://xkcd.com/1172/)) changes have occurred between the version I'm on and the version I'm upgrading to.

Pretty simple stuff, but let's see how the difficulty of completing this task differs between Github Releases and a CHANGELOG.md file.

### Github Releases

We navigate to our repository's Releases page, where we can see the 10 most recently released versions. Most of them are patches, so we only get one minor release on the whole page.

![](/images/changelog/releases-index.png)

This isn't the version we're upgrading from though, obviously. It's ok though, there's a search bar, so we can just search for my exact version! Except...

![](/images/changelog/releases-bad-search.png)

Nope, the Releases search bar behaviour is... *unintuitive*. Passing the exact Release tag string doesn't guarantee it comes up first, or even on the first page. Adding quotation marks doesn't either. It [turns out](https://docs.github.com/en/repositories/releasing-projects-on-github/searching-a-repositorys-releases#search-syntax-for-searching-releases-in-a-repository) the magic incantation is `tag:blah`:

![](/images/changelog/releases-search-better.png)

Great! We have the version we're running now. Now we just need to page forwards through the newer versions. Except, that is not possible from this view:

![](/images/changelog/releases-search-no-pager.png)

So I guess we'll have to return to the index page, and page backwards from the latest version all the way to the version we are currently on after all. In batches of 10. Fun.

![](/images/changelog/releases-pager.png)

Well this is awful. Oh, at least there's a `&page=X` query param. We can use that to skip back and forth through pages homing in on the page featuring our version. We go to `page=3` and the last version is too new. On `page=8` the first version is too old. How about `page=5`? Last version too new again... **Somehow we have gone from upgrading a library to executing a human powered binary search** 🤢.

Eventually we find the right page, and can now start paging forwards through the newer versions, studying the associated notes. Even now though we run into an annoying behaviour we need to be aware of. Since releases are ordered chronologically by **publish** date there may be releases "between" versions in the list that do not apply to us because they are not present in the version we are running, because they are from another branch:

![](/images/changelog/releases-out-of-order.png)

Mercifully it is possible to suppress pre-releases with `prerelease:false` in the search UI. However, this doesn't solve the issue where older major versions are maintained alongside newer versions as is common for bigger dependencies like languages and frameworks. In those cases, you just have to skip past these versions.

All in all, quite the ordeal to complete a seemingly simple task. So how does this *modern clean solution* compare to the *ancient archaic text file* approach?

### CHANGELOG.md

We navigate to our repository's CHANGELOG.md in its root directory, where we can see all versions ever. Next, we `Ctrl+F` for our version, type its name into the browser find in page UI, and we are instantly scrolled down to it. If you prefer mice to keyboards, we also have a list of jump links to every version.

![](/images/changelog/changelog-ctrl-f.png)

It's not uncommon to have the entire repository's history in this single page. [React](https://github.com/facebook/react/blob/4ddff7355f696ec693c5ce2bda4e7707020c3510/CHANGELOG.md#030-may-29-2013) for example has **11 years of history** on this one page.

And to see changes between the upgraded version and ours? Scroll up. That's it 😄. By virtue of being branch specific, changelogs sidestep the aforementioned pre-release problem, because the repository's trunk branch changelog only contains changes that have landed on the trunk. If you _are_ a pre-release tester, the release branch's changelog has what you need.

----

I think the UX chasm between these two workflows is obvious, and that is why I am surprised and concerned to see CHANGELOG.md falling out of fashion.

**If you are a Github employee**, please, improve this experience! If Releases are to replace changelogs, as I think they reasonably could, their UX needs to be _better_ than changelogs, not worse. A UI to see changelogs between releases X and Y would be a game changer.

**If you are a repo maintainer** that has migrated entirely to Github Releases, please consider **maintaining a CHANGELOG.md** also. If that's too much effort consider **generating a CHANGELOG.md from your existing Github Releases**. Thanks to the [**rhysd/changelog-from-release**](https://github.com/rhysd/changelog-from-release) this is a pretty simple thing to do. It can even be configured to ignore certain releases, enabling you to e.g. exclude pre-releases from your trunk changelog.

Thanks for reading, and above all, much love to all open source maintainers out there! ❤️
