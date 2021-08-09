---
title: "The Golang.org URL redirector"
---

I always thought it was interesting that the [Go project](https://golang.org) always uses "vanity" URL redirectors to link to things like GitHub issues and GitHub wiki pages and CLs, which I thought would be pretty static things.
Can we figure out what these redirectors do, and what they're _meant_ to do?
Is there something more to it than vanity?

Within commit messages, issues and PRs, you'll see humans and bots make references to GitHub issues through https://golang.org/issue/1234, and Gerrit changelists through https://golang.org/cl/1234, and so on.
This was the bit that puzzled me, why not just link to the actual issue or CL?

There are also another category of links like https://golang.org/s/sometext, which makes more obvious sense -- you want to use a redirector/shortener in those cases to ensure a stable URL for published links, for content that might get moved around in future.

(This post is not really a blog post since there's no real narrative, but more of publishing notes online for search engines to find, I guess.)

## Current state

Right now, these redirects exist in the [golang.org/x/website/internal/redirect package](https://go.googlesource.com/website/+/1459be3397876c3d750be16d4063be347520681a/internal/redirect/redirect.go), where we can marvel at the full list of redirects and other special handlers.
They can be grouped into a few categories:

* Legacy path redirects
    * Packages renamed between r60 and go1, e.g. `csv` to `encoding/csv`, `json` to `encoding/json`
    * Commands renamed between r60 and go1, e.g. `goinstall` and `gotest` to `go` (who knew!)
* Prefix redirects
    * (Whatever comes after these prefixes in the requests is used to construct a link to a specific URL elsewhere, e.g. `/issue/1234` to `https://github.com/golang/go/issues/1234`.)
    * `/issue/`
    * `/issues/`
    * `/play/`
    * `/talks/`
    * `/wiki/`
* Redirects
    * Misc shortcuts like `/build`, `/change`, `/cl`, `/play`, `/design` and old docs that were moved to the blog or wiki or tour
* Special cases
    * `/src/pkg/` for the Go 1.4 removal of `pkg/` dir, http://golang.org/s/go14nopkg
    * `/cl/` to disambiguate between Rietveld CL and Gerrit CL
    * `/change/` to link to specific git commits, after converting Mercurial to Git revisions
    * `/design/` to link to specific design docs, after appending `.md` to the name

Separate from all this is the `/s/` handler, which seems to be a more "traditional" or "dynamic" URL shortening service, backed by Cloud Datastore.
We can't see what links exist here, since it's not in code, so assembling a list would probably mean grepping through the codebase and issues for `/s/` refs.

While first three sections contain some relatively standard backward-compatible type of things that might historically have been done with `.htaccess` files in httpd, the last section is the most interesting.

The **`/change/` handler** loads a map of Mercurial to Git revisions from a file somewhere, which is quite clever, since it's presumably static and as part of the source repo migration you'd have ensured that all old commits exist in the new repo.

You can test the Mercurial to Git conversion by going to [an old Rietveld CL](https://codereview.appspot.com/13722043), finding a link like `https://code.google.com/p/go/source/detail?r=dc6035c8db3f&repo=tools`, copy the `r=` value out and append it like so: `https://golang.org/change/dc6035c8db3f`.
[That URL](https://golang.org/change/dc6035c8db3f) will send you to the converted git commit on the new git repo.

The **`/cl/` handler** has a fair bit of logic to determine whether a given integer is a Rietveld CL, a Gerrit CL, or both.
Presumably they couldn't migrate past CLs from Rietveld to Gerrit, but yet they decided to continue using the same prefix for the new Gerrit CLs:

* The Rietveld CL identification logic is a combination of assuming that all numbers above 300k are Rietveld CLs (apparently they go up into the millions!), plus a static whitelist of the thousand or so Rietveld CLs that sparsely exist between 152,046 and and 299,999.
* If a number is identified to be a Rietveld CL, the Gerrit CL identification logic is triggered, which actually queries Gerrit and stores the result in an in-memory cache.
* If it's ambiguous, a HTML page that offers links to both is returned instead of a redirect, e.g. https://golang.org/cl/157099

This actually feels like quite a lot of incremental effort over the past 6-7 years to retain old URLs!

## Historical references

Rob Pike's Nov 2014 [announcement on golang-dev](https://groups.google.com/forum/#!topic/golang-dev/sckirqOWepg) of the move from Mercurial on code.google.com to Git on GitHub ([HN discussion](https://news.ycombinator.com/item?id=8605204)): 

> P.S. For those keeping score, this will be Go version control system number four. Go development started in Subversion, moved to Perforce, then Mercurial, and soon to Git.

(I suppose we should be happy they didn't try to write a VCS and issue tracker in Go for Go development?)

**Major changes to this redirect code (old to new):**

| Link | Description |
|---|---|
| [Rietveld CL 13722043](https://codereview.appspot.com/13722043) | Initial add of redirects |
| [Rietveld CL 13356047](https://codereview.appspot.com/13356047) | Add redirects for old content to new content |
| [Rietveld CL 187920043](https://codereview.appspot.com/187920043) | Switch from code.google.com to github.com and Rietveld to Gerrit URLs |
| [Gerrit CL 1567](https://go-review.googlesource.com/c/tools/+/1567/) | Add support for the Mercurial to Git map |
| [Issue 26871](https://golang.org/issue/26871) | Send `/design` to Gerrit instead of GitHub, partly because it offers nice printable HTML output |
| [Issue 28836](https://golang.org/issue/28836) | Gerrit CL numbers started to collide with Rietveld numbers, add disambiguation page |

The old Rietveld CLs don't seem to have references to an issue tracker, or I'd point to that instead.
Maybe it's all on a mailing list somewhere?

The `/cl/` convention turns out to have been around for really long.
The commit for issue #1 mentions `http://go/go-review/1017003`, but from issue #2 onwards the commit messages point to `https://golang.org/cl/152048`, so the `/cl/` convention has been around since the [very early days of Nov 2009](https://github.com/golang/go/commit/484f46daea9a44afd9fc0ea90b2172dfa524d9bb).

**Git history/log (old to new):**

* [tools cmd/godoc/handlers.go](https://go.googlesource.com/tools/+log/d20f86cc8e30eb09a90497e5a01c4f97ada5b445/cmd/godoc/handlers.go)
* [tools godoc/redirect/](https://go.googlesource.com/tools/+log/0a99049195aff55f007fc4dfd48e3ec2b4d5f602/godoc/redirect)
* [website internal/redirect/](https://go.googlesource.com/website/+log/55955d3c0752ab9fb9163f43a5106f3275ca4459/internal/redirect)