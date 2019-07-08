---
title: "Mozilla's Server Side TLS 5.0"
date: 2019-07-08
draft: false
---
I got distracted into yak-shaving about TLS cipher suites today when I noticed that Mozilla's [Server Side TLS](https://wiki.mozilla.org/Security/Server_Side_TLS) document had been updated -- just 10 days ago, it turns out -- so I figured I'd try and write down some of what I learnt.

I'm going to assume you know that TLS is.
Part of the TLS protocol involves some form of negotiation between the server and client about which set of ciphers should be used for the connection.
Picking what ought to be on this list, and in what order, can be more complicated than you might expect!
There are something like 200-300 cipher suites.
The 'practical' way of doing it might be to trial-and-error until Nessus stops complaining (and nobody opens support cases), but that doesn't seem ideal.

The Mozilla Operations Security team (and now also their [Enterprise Information Security (Infosec)](https://infosec.mozilla.org/) team?) has been maintaining this Server Side TLS document that provides config recommendations for servers.
It seems to have started sometime in 2013, perhaps primarily meant for Mozilla sites, but at some point people started looking at it as a reference.

> You might also have seen the Infosec team's [OpenSSH guidelines](https://infosec.mozilla.org/guidelines/openssh.html) at some point, if you're a sysadmin.

Other resources on these include lots of random outdated tutorials about getting A+ on SSL Labs' SSL Server Test, and the [cipherli.st](https://cipherli.st/) site.
The actual SSL Labs grading criteria is probably what gets updated most often though, as covered in their [SSL Labs Changelog](https://community.qualys.com/docs/DOC-5737-ssl-labs-changelog).

> If you're interested in TLS and internet PKI things, Feisty Duck runs the excellent [Bulletproof TLS Newsletter](https://www.feistyduck.com/bulletproof-tls-newsletter/). They don't email very frequently, at most once a month, and my mailbox actually confirms that's held true for the past two years.

(While researching this I found that I'd actually gone down this rabbit hole and filed a KIV issue somewhere slightly over a year ago, but this seems like a good time to revisit it given that TLS 1.3 support has been slowly rolling out.)

## Intermediate configuration changes

Intermediate is the config that most people should care about.

You shouldn't be using Modern unless you control all client and server endpoints, and hopefully you don't need to use Old unless you need to support the very old software versions listed.

However, I'm comparing old Modern config with the current Intermediate config, because my existing configs are closest to those two, since I haven't had to address SSLv3 or TLSv1.0/1.1 compatibility for quite a long time.

The details are at the [current 5.0 page](https://wiki.mozilla.org/Security/Server_Side_TLS#Intermediate_compatibility_.28recommended.29) and the [archived 4.0 page](https://wiki.mozilla.org/Security/Archive/Server_Side_TLS_4.0#Intermediate_compatibility_.28default.29), but the cipher suites are partially reproduced here since it's the most interesting bit:

**4.0 Modern** cipher suite list (Feb 2016, [mozilla/server-side-tls#97](https://github.com/mozilla/server-side-tls/pull/97)):

```
ECDHE-ECDSA-AES256-GCM-SHA384  TLSv1.2
ECDHE-RSA-AES256-GCM-SHA384    TLSv1.2
ECDHE-ECDSA-CHACHA20-POLY1305  TLSv1.2
ECDHE-RSA-CHACHA20-POLY1305    TLSv1.2
ECDHE-ECDSA-AES128-GCM-SHA256  TLSv1.2
ECDHE-RSA-AES128-GCM-SHA256    TLSv1.2
ECDHE-ECDSA-AES256-SHA384      TLSv1.2
ECDHE-RSA-AES256-SHA384        TLSv1.2
ECDHE-ECDSA-AES128-SHA256      TLSv1.2
ECDHE-RSA-AES128-SHA256        TLSv1.2
```

**5.0 Intermediate** cipher suite list (Jun 2019, [mozilla/server-side-tls#255](https://github.com/mozilla/server-side-tls/pull/255)):

```
TLS_AES_128_GCM_SHA256         TLSv1.3
TLS_AES_256_GCM_SHA384         TLSv1.3
TLS_CHACHA20_POLY1305_SHA256   TLSv1.3
ECDHE-ECDSA-AES128-GCM-SHA256  TLSv1.2
ECDHE-RSA-AES128-GCM-SHA256    TLSv1.2
ECDHE-ECDSA-AES256-GCM-SHA384  TLSv1.2
ECDHE-RSA-AES256-GCM-SHA384    TLSv1.2
ECDHE-ECDSA-CHACHA20-POLY1305  TLSv1.2
ECDHE-RSA-CHACHA20-POLY1305    TLSv1.2
DHE-RSA-AES128-GCM-SHA256      TLSv1.2
DHE-RSA-AES256-GCM-SHA384      TLSv1.2
```

As a reminder, the order of components in these TLSv1.2 IANA cipher suite names is Key Exchange (Kx), Authentication (Au), Encryption (Enc), MAC (Mac).

What are the key differences?

1. TLS 1.3 is enabled, with 3 out of 5 of the default OpenSSL cipher suites
1. All four non-AEAD cipher suites have been removed.
1. Re-ordered to prefer AES-128 over AES-256 over ChaCha20.
1. There are these two fallback-looking options tacked on at the end, which are Kx=DHE instead of ECDHE, but only for Au=RSA, which is oddly specific.
1. Curves are changed to prefer X25519 first, and drop secp521r1
1. Cipher negotiation is now based on client's preference, not server's.

Why those differences? The answer is spread over several GitHub issues (primarily [mozilla/server-sde-tls#178](https://github.com/mozilla/server-side-tls/issues/178)) but it's a really tedious read at times while they discuss the Modern config so here's the tl;dr:

1. Of course TLS 1.3 should be enabled, it's 2019! It's not very clear the two AES-CCM cipher suites are excluded probably because they're meant more for embedded systems and not general use, so there probably won't be any browser support.
1. There are opinions that CBC (perhaps particularly AES-CBC) has problems, using only AEAD primitives avoids that.
1. Previously ChaCha20-Poly1305 was prioritised to deal with mobile hardware that doesn't support AES-NI. Now they choose to assume that the client is aware of whether it has AES acceleration and will decide to prioritise ChaCha20-Poly1305 or not (this requires #6). AES-128 is now recommended over AES-256 for performance reasons, on the basis that there's no significant security advantage to using AES-256.
1. The two DHE-RSA-AES-GCM fallbacks are added specifically for IE 11 on Windows 7 and 8 (but not 10). That particular combination only supports AES-GCM with DHE-RSA or ECDHE-ECDSA, but not ECDHE-RSA, so servers using RSA certs wouldn't be able to negotiate a connection without those fallbacks.
1. They actually doesn't discuss this much, it seems to be a given that it's superior to NIST curves. secp521r1 was mentioned as being rarely used and hence potentially problematic.
1. Required for #3.

## Misc observations

- ECDSA certs continue to be recommended/encouraged, but this may be a little tougher for most folks.
  I know that I haven't considered it at all.
  For one, most ACME (Let's Encrypt) clients currently only do RSA; the exceptions are primarily Caddy and Lego.
  It can also be a little tricky to configure dual ECDSA/RSA certs in your favourite server.
- One opinion was that if you only do ECDHE/DHE, there's no value in having your cert's key be of RSA >2048 bits; going above that key length is only _potentially_ valuable with static RSA key exchange.
- One thing that isn't new but somehow surprised me was the bit about recommending the pre-defined DH group `ffdhe2048` defined [RFC 7919](https://tools.ietf.org/html/rfc7919), instead of the usual `openssl dhparam` incantation.
  I suppose having an audited thing is a good idea.
