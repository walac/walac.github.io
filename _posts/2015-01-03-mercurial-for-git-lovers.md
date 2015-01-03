---
title: "Mercurial for git lovers"
category: VCS
comments: true
tags: [mercurial]
---

So far I have been a heavy [git](http://git-scm.com/) user, but since I joined
[Mozilla](http://www.mozilla.org) I have given
[Mercurial](http://mercurial.selenic.com/) a try. I must
say that migrating from [Subversion](https://subversion.apache.org/) to git
was far less painful than from git to Mercurial. I think there a few reasons
to that:

* When I started to learn git, I started it with
[PyUSB](http://walac.github.io/pyusb), a small personal project that was on its
youth at that time. This means I needed just the most basic commands to start
using it. When I started on Mercurial, I was working on
[Gecko](https://developer.mozilla.org/en-US/docs/Mozilla/Gecko), a very big and
complex project, with thousands of contributors, so I had to learn
more advanced commands from day one.

* When you install git, you have all you need, all commands you need are there.
Mercurial, in the other hand,
[operates in a very different way](http://gregoryszorc.com/blog/2013/05/12/thoughts-on-mercurial-%28and-git%29/).
It just ships with some basic commands, delegating most of the more advanced
stuff to extensions. So you have to dig to find the extensions that enable the
Mercurial approach to commands you love in git.

* There is nothing like git branches parallel in Mercurial, and 9/10 of git users'
workflow rely heavily on branches (more on that later).

Most of the day to day commands in Mercurial are quite similar to git, like
[clone](http://selenic.com/hg/help/clone) and
[commit](http://selenic.com/hg/help/commit). Some others have very different names,
like [git revert](http://git-scm.com/docs/git-revert), which in Mercurial
is called [backout](http://mercurial.selenic.com/wiki/Backout).
[hg revert](http://selenic.com/hg/help/revert) is used to discard non-committed
changes in your repository.

I prepared a list of some Mercurial extensions that will make your life easier
when coming from git. These extensions will make you feel more comfortable
while moving from git to Mercurial.

git feature                                                                                     | Mercurial extension
------------------------------------------------------------------------------------------------|---------------------------------------------------------------------
[color](http://git-scm.com/book/en/v2/Customizing-Git-Git-Configuration#Colors-in-Git)          | [Color](http://mercurial.selenic.com/wiki/ColorExtension)
[git rebase](http://git-scm.com/docs/git-rebase)                                                | [Rebase](http://mercurial.selenic.com/wiki/RebaseExtension)
[Interactive rebase](https://www.atlassian.com/git/tutorials/rewriting-history/git-rebase-i)    | [Histedit](http://mercurial.selenic.com/wiki/HisteditExtension)
[git stash](http://git-scm.com/docs/git-stash)                                                  | [Shelve](http://mercurial.selenic.com/wiki/ShelveExtension)
[git clean](http://git-scm.com/docs/git-clean)                                                  | [Purge](http://mercurial.selenic.com/wiki/PurgeExtension)
[git add -i](https://www.kernel.org/pub/software/scm/git/docs/git-add.html)                     | [Record](http://mercurial.selenic.com/wiki/RecordExtension)
[gitk](http://git-scm.com/docs/gitk)                                                            | [Hgk](http://mercurial.selenic.com/wiki/HgkExtension)
[\_\_git_ps1](http://fedoraproject.org/wiki/Git_quick_reference#Display_current_branch_in_bash) | [Prompt](http://mercurial.selenic.com/wiki/PromptExtension)
[git cherry-pick](http://git-scm.com/docs/git-cherry-pick)                                      | [Transplant](http://mercurial.selenic.com/wiki/TransplantExtension)
[git send-email](http://git-scm.com/docs/git-send-email)                                        | [PatchBomb](http://mercurial.selenic.com/wiki/PatchbombExtension)
[git am](http://git-scm.com/docs/git-am)                                                        | [Mbox](http://mercurial.selenic.com/wiki/MboxExtension)
[Topic branches](http://fiji.sc/Git_topic_branches)                                             | [Bookmarks](http://mercurial.selenic.com/wiki/BookmarksExtension)
[Paged log](http://git-scm.com/docs/git-log)                                                    | [Pager](http://mercurial.selenic.com/wiki/PagerExtension)

This [link](http://mercurial.selenic.com/wiki/GitConcepts) has a lot more
information about differences between git and Mercurial, including Mercurial
counterparts for several git actions.

Mercurial comes with an important feature that there is no equivalent
in git at all, called [phases](http://mercurial.selenic.com/wiki/Phases).
I mention it here because you may have problems with it if you (like me)
do a lot of rebase and history editing. Every commit you push or pull
to/from a remote repository is considered `public`, which makes it
immutable. This happens all the time to me because I often push commits to
[Mozilla Try](https://wiki.mozilla.org/ReleaseEngineering/TryServer)
and [reviewboard](https://reviewboard.mozilla.org). Thus, I generally
face errors like this:

```
    hg rebase -d bookmark0
    abort: can't rebase immutable changeset 35c25b97fca2
    (see hg help phases for details)
```

That's because I have pushed this commit and Mercurial made it
`public` (aka immutable). The solution for this is make it
a `draft` commit again:

```
    hg phase --draft --force bookmark3
```

I find this feature very annoying and I think it would be better
implemented as an extension. It sounds Java telling me what I
can and can't do. You can disable making commits public after
a push by adding these entries in your
[hgrc](http://www.selenic.com/mercurial/hgrc.5.html) file:

```
    [phases]
    publish = False
```

### Special note on Bookmarks

Since Mercurial version 1.8, Bookmarks is now part of Mercurial core. It is
often advertised as *git branches on Mercurial*. **It is not**! The most
difficult part for me was to understand that Mercurial has no branch
support like git. Period. Bookmarks is just a hack that tries
to mimic topic branches, but, technically, a bookmark is not a branch, it is
more like a tag. I am not going to explain how bookmarks work, you can learn more
on it [here](http://mercurial.aragost.com/kick-start/en/bookmarks/).

The most irritating thing regarding Bookmarks is that changing the history in
one bookmark can affect other bookmarks. In some cases, when you
create multiple heads descending from a bookmark, you cannot edit its
history at all. Let's try an example. Imagine you fix a bug and submit it
for review. While you wait for feedback, you create more bookmarks to work
on other product features. You eventually end up with a tree like this:

```
    @ bookmark 3
    |
    | o bookmark 2
    |/
    |
    | o bookmark 1
    |/
    |
    o bookmark 0 (bug 137463)
```

The `@` indicates the commit representing the current directory.

After a while you receive feedback for `bug 137463` and you are requested
some changes. You then move back to `bookmark 0`:

```
    o bookmark 3
    |
    | o bookmark 2
    |/
    |
    | o bookmark 1
    |/
    |
    @ bookmark 0 (bug 137463)
```

And start to apply the requested changes, making a new commit:

```
    o bookmark 3
    |
    | o bookmark 2
    |/
    |
    | o bookmark 1
    |/
    |
    | @ bookmark 0 (bug 137463)
    |/
    |
    o commit 1 (original bug 137453 commit)
```

`bookmark 0` moved to the new commit.
What you want now is to squash `commit 1` and `bookmark 0`, but you can't,
because you will mess up bookmarks 1-3. One solution would be to rebase
bookmarks 1-3 on top of `bookmark 0`:

```
    o bookmark 3
    |
    | o bookmark 2
    |/
    |
    | o bookmark 1
    |/
    |
    @ bookmark 0 (bug 137463)
    |
    o commit 1
```

You can now `histedit` from `commit 1` and squash it with `bookmark 0`, right?
Wrong! This happens because the structure you see is just an illusion from
branching point of view. In git, when you modify a branch, it doesn't affect
descendant branches. In Mercurial this is not true simply because all the commits
**are in the same branch**.

### Conclusion

If your git workflow relies heavily on git branching capabilities and history
editing, you will have some trouble to adapt yourself to Mercurial.

If I could request just one feature to Mercurial developers, that would be
lightweight branches like git. That would give Mercurial a big boost.

There are other problems I had with bookmarks regarding remote repositories,
but I will save that for a future post.
