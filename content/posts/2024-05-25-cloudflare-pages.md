+++
title = "Migrating to Cloudflare Pages"
+++

After the "[Netlify just sent me a $104K bill for a simple static site](https://old.reddit.com/r/webdev/comments/1b14bty/netlify_just_sent_me_a_104k_bill_for_a_simple/)" (link goes to r/webdev) incident about three months ago, where someone's site had very many downloads of a 3 MB mp3 file and was hit with the $104k (discounted to $5k) bandwidth overage bill, I've had moving off Netlify on my to-do list.

This site is now deployed on Cloudflare Pages.

## Deploying to Cloudflare Pages

Zola is no longer supported out of the box on the Cloudflare Pages' current v2 build system.
Cloudflare Pages' [Zola framework guide](https://developers.cloudflare.com/pages/framework-guides/deploy-a-zola-site/) instructions simply do not work, the build fails with an `zola: not found` error.

Fortunately, someone has documented some workarounds on the [Zola deployment docs for Cloudflare Pages](https://www.getzola.org/documentation/deployment/cloudflare-pages/#zola-not-found).
The magical incantation of setting the `UNSTABLE_PRE_BUILD` environment variable with the value `asdf plugin add zola https://github.com/salasrod/asdf-zola && asdf install zola 0.18.0 && asdf global zola 0.18.0` made the build work.
We can only hope that the build system doesn't deprecate this undocumented environment variable.

An issue has been filed ([cloudflare/pages-build-image#3](https://github.com/cloudflare/pages-build-image/issues/3)) since July 2023, not long after the v2 build system started rolling out in May 2023, but there has apparently not been any movement on improving v2 to get closer to feature parity with v1, nor improving their docs.
Cloudflare Pages overall just doesn't seem to have moved much in the last few quarters, looking at the docs changelog.

## Setting up custom domains

I followed the [custom domains docs](https://developers.cloudflare.com/pages/configuration/custom-domains/) and added `www.cflee.com`.
That seemed to go fine.
As my `cflee.com` zone was already on Cloudflare in a full setup, it automatically offered to update the existing CNAME record to point to the `<name>.pages.dev` project DNS name, and enable proxying.

The docs have an aside about ensuring that you set it up from the Pages end.
Don't skip directly to adding/updating the CNAME record on the DNS end.

## Migrating redirects

I had some redirects set up to redirect requests made to various DNS names to the canonical hostname, `www.cflee.com`.

Previously, all the other DNS names pointed to the same Netlify site, where each was configured as a custom domain, and then redirects were done in the Netlify config file.
Netlify also automatically configured the root/www domains together.

When I tried replicating this setup on Cloudflare Pages, I found that the [`_redirects` config file](https://developers.cloudflare.com/pages/configuration/redirects/) only supports file paths, so nothing to do redirects across hostnames.

After some poking around the Cloudflare community forums, I found that it could be done using Page Rules.

Well, no, there's a big banner saying that Page Rules are deprecated.
After more clicking around the docs for the four successor rule types, you land on the [redirect rule docs](https://developers.cloudflare.com/rules/url-forwarding/)... which are quite inscrutable.
Even after you figure out you want a [dynamic URL redirect](https://developers.cloudflare.com/rules/url-forwarding/single-redirects/settings/#dynamic-url-redirect), you then still have to figure out the expression using fields and functions... 

The [example in the docs](https://developers.cloudflare.com/rules/url-forwarding/single-redirects/examples/) are OK, but this [cheatsheet on the forums](https://community.cloudflare.com/t/redirect-rules-cheat-sheet/508780) is extra helpful.

Eventually what I wanted seemed to be:

- Hostname equals `cflee.com`
- Dynamic URL redirect to `concat("https://www.cflee.com", http.request.uri.path)`, status code 301, query string preserved

Repeat this on the other domain for the subdomains there.

You only get 10 of these redirect rules per domain on the free plan, but at least you can set up some 'OR' conditions on the matcher expression to match multiple hostnames, or even do a "contains" operator on the matcher.

Now, to get these redirect rules working, the DNS records still need to be created on Cloudflare and proxying enabled.
But all DNS records need to be pointed somewhere, and there is no option in Cloudflare's console to just have a placeholder record that is only to be handled by Cloudflare proxying.

In my case, I had the existing CNAME records pointing at Netlify and could just enable the proxying and be done with it.
But it seemed bad practice to leave the DNS records pointing to Netlify, as in the event of the redirect rule or proxying being accidentally disabled, someone could take over the dangling subdomain at Netlify.

Different parts of the Cloudflare docs have slightly different recommendations on what the DNS records should be pointed at.

- [Cloudflare DNS docs](https://developers.cloudflare.com/dns/troubleshooting/faq/#what-ip-should-i-use-for-parked-domain--redirect-only--originless-setup) recommend using `192.0.2.0` and `100::` for this exact case of proxied DNS names for Cloudflare Rules or Cloudflare Workers.
- [Cloudflare Page Rules docs](https://developers.cloudflare.com/rules/page-rules/#page-rules-require-proxied-dns-records) recommend using `192.0.2.1`, `2001:DB8::1`, or `domain.example`.
- [Cloudflare Pages howto](https://developers.cloudflare.com/pages/how-to/www-redirect/) (I clearly only just discovered this afterwards while writing this post) says that it _must_ be either `192.0.2.1` or `100::`.

I decided to go with a CNAME to `domain.example` since that would be most obvious to me that it's an invalid record, to avoid future me scratching my head over why I have some records pointing to a `192.0.x.x` address.

This whole section was evidently even more frustrating than the build system issue.
For that, some googling quickly brought me to the relevant pages with a fix to copy.
These redirects took a fair bit more work to figure out, given that you have to wield the DNS / proxying side of things carefully to patch this gap, and the experience could have been smoother with better (and more consistent) docs.

## Limits

It's worth looking at the [Cloudflare Pages limits docs](https://developers.cloudflare.com/pages/platform/limits) prior to starting on migrating anything.
It generally seems reasonable, though there are some hard limits worth noting, like the build count per month (probably where the config for only having certain branches trigger builds/deploys can be helpful), number of custom domains, max file size, number of files, number of header rules, and redirect rules.

The [Serving Pages docs](https://developers.cloudflare.com/pages/configuration/serving-pages/) is also a necessary need, in case your site might be affected by certain limitations.
In particular, the behaviour of redirecting requests for `.html` paths to strip off the `.html` is potentially very surprising, and there is no documented workaround.
