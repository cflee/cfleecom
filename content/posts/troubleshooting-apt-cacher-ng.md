---
title: "Troubleshooting corrupt apt downloads through apt-cacher-ng"
date: 2019-11-23
draft: false
tags:
- linux
---

This was an interesting problem that isn't too environment-specific, so I thought it might be interesting to write up.

**tl;dr** -- if you are using apt-cacher-ng and getting corrupt Release/Packages files that contain a mix of stale and fresh data, check if the upstream server fully supports HTTP/1.1 range requests, and if it doesn't, set `VfileUseRangeOps:0` on apt-cacher-ng.

## Context

Within internal network environments, apt-get on hosts can be set to use an [apt-cacher-ng](https://www.unix-ag.uni-kl.de/~bloch/acng/) instance as a caching proxy, via the `Acquire::http::proxy` directive.
This allows hosts' requests for recently requested packages to be served quickly, which is particularly useful when patching a whole bunch of hosts.

As a pull-through cache that does APT-aware cleanups, only the package files that have been requested _and_ are still current (in not having been superseded by a newer package) are retained on disk, which can save a lot of storage compared to mirroring the full Ubuntu/Debian repositories using apt-mirror.

APT just treats it as a regular HTTP proxy, so there's no special protocol to communicate with it.
There is some special handling required to access external repos over HTTPS, via apt-cacher-ng's ["tell-me-what-you-need"](https://www.unix-ag.uni-kl.de/~bloch/acng/html/howtos.html#ssluse) method, but that's not really material to this issue.

> Whether it's necessary or appropriate to retrieve checksummed-and-PGP-signed repo metadata and package files over HTTPS is really best left to another discussion...

## Symptoms

After each release of [Fluent Bit](https://fluentbit.io), APT on all our hosts would start complaining about receiving an invalid InRelease file, and/or invalid hash sum (checksum) on Packages.bz2.
Both of these files are metadata files in the [Debian Repository Format](https://wiki.debian.org/DebianRepository/Format). 
The [Release file](https://wiki.debian.org/DebianRepository/Format#A.22Release.22_files) is the information root of the distribution that contains checksums for the indices, while the [Packages index](https://wiki.debian.org/DebianRepository/Format#A.22Packages.22_Indices) contains metadata and checksums for the actual package files.

Strangely, the issue was repeatedly appearing only with the Fluent Bit repository.
Other repositories like [Docker Engine](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-using-the-repository) that were accessed over HTTPS for whatever reason weren't impacted, so it's not some HTTPS-specific quirk in apt-cacher-ng.

The problem could be worked around temporarily by deleting the corrupt files from apt-cacher-ng's cache, but the issue recurs at the next repository update.

## Analysis

### Client-side errors

For the InRelease files, the specific error from APT was a complaint that it could not separate out the PGP signature from the data, for signature verification.

> The difference between Release and InRelease is that instead of having separate Release and Release.gpg files side by side, InRelease has the GPG signature appended to it as an _In_-line signature.
> This avoids the problem of getting a signature file that doesn't match the Release file, if the repository is updated while a client is downloading the two files.

Inspecting the downloaded file that APT rejected, it has all the old data from the previous file, except that the final line for the ASCII Armor Tail Line ([RFC 4880](https://tools.ietf.org/html/rfc4880#section-6.2)) had six ending dashes instead of the expected five.
That was the direct cause of the signature verification on the file, apart from it being the wrong file entirely.

Interestingly, The corrupt file with that sixth dash was the "correct" size at 4867 bytes; it is one byte longer than the previous version of the file at 4866 bytes, because the day in the date field went from single-digit (`4`) to double-digit (`11`).

### Server-side corruption

Increasing apt-cacher-ng's logging verbosity with `Debug:7` did not really help, and turning it up any further (perhaps beyond `Debug:3`) required a compile-time flag to include more debug statements.

After chasing down several other red herrings, the breakthrough came when an apt-cacher-ng instance in another environment with an uncorrupted previous copy of the `main/binary-amd64/Packages.bz2` file was found, and from there copies were made of the cached files before and after an APT client made a request for the file.

Using [HexFiend](https://ridiculousfish.com/hexfiend/) (though diffing the text output of `od` or `xxd` would work too) to inspect a binary diff of the corrupt file and a fresh intact file downloaded directly from upstream yielded nothing interesting, as it was pretty much all different, except the tail bit of the file was fully intact.

But upon diffing the intact old file and the corrupt file, it was immediately obvious that the corrupt file was composed of the old file + the last 192 bytes or so of the new file, suggesting that bytes from the new file were appended to reach the length of the new file.
That is consistent with the sixth dash seen in the InRelease file, where the actual newer version was one byte longer, and the last byte was a dash.

### apt-cacher-ng behaviour

Some spelunking into the [apt-cacher-ng source code](https://salsa.debian.org/blade/apt-cacher-ng/tree/upstream/3.1) later, it turns out that it has a concept of "volatile" files, that are handled differently from "static files". 
"Volatile" means that the contents of a given file will vary over time after it's initially published, like all the Release and Packages metadata which are updated with new checksums, in contrast with "static" where the package files that are identified by a specific version number in the filename are expected to never change.
Apt-cacher-ng will always check for updates for "volatile" files, and never for "static" files that are already present in the cache.

When checking for updates on "volatile" files that exist in its cache, apt-cacher-ng makes use of the HTTP/1.1 range requests features, as defined currently in [RFC 7233 sections 3.1-3.2](https://tools.ietf.org/html/rfc7233#section-3), and previously in RFC 2616 sections [14.35](https://tools.ietf.org/html/rfc2616#section-14.35) and [14.27](https://tools.ietf.org/html/rfc2616#section-14.27).

It adopts an APT-inspired technique of making a HTTP request that should return either (i) the last byte of the file if the file is up to date, or (ii) all bytes if the file is has been updated since the specified timestamp.

An request/response might look like this, with some headers elided for brevity:

```
> GET /linux/ubuntu/dists/bionic/InRelease HTTP/1.1
> Host: download.docker.com
> Range: 64441-
> If-Range: Sat, 19 Oct 2019 01:07:31 GMT
>
< HTTP/1.1 206 Partial Content
< Content-Type: binary/octet-stream
< Content-Length: 1
< Last-Modified: Sat, 19 Oct 2019 01:07:31 GMT
< Content-Range: bytes 64441-64441/64442

> GET /linux/ubuntu/dists/bionic/InRelease HTTP/1.1
> Host: download.docker.com
> Range: 64441-
> If-Range: Sat, 19 Oct 2019 00:00:00 GMT
>
< HTTP/1.1 200 OK
< Content-Type: binary/octet-stream
< Content-Length: 64442
< Last-Modified: Sat, 19 Oct 2019 01:07:31 GMT
```

The idea [as expressed in apt-cacher-ng source](https://salsa.debian.org/blade/apt-cacher-ng/blob/upstream/3.1/source/dlcon.cc#L378-389) is that it wants to avoid receiving a 416 Range Not Satisfiable error, and would rather "waste" the transfer of a single byte if the cached file is up-to-date, than to handle a 416 correctly.

I don't really know why APT used to do that over `If-Modified-Since:`, but this single-byte thing seemed to be removed and replaced in APT with proper 416 response handling back in Sep 2013 in [78c72d0c](https://salsa.debian.org/apt-team/apt/commit/78c72d0ce22e00b194251445aae306df357d5c1a) (v0.9.12).
Now the comments in APT seem to indicate that conditional range requests should only be used to resume partial downloads, and checking freshness should be accomplished using `If-Modified-Since:`, instead of this `If-Range:` header.

### Server behaviour

The problem arises where the HTTP server is not fully compliant with the spec, and acts on the `Range:` header while ignoring the `If-Range:` header.
In that case, given a conditional range request, it will always respond with either a 206 Partial Content (with some subset of bytes), or 416 Range Not Satisfiable (due to the file being shorter than the client thinks it is), and never with a 200 OK (with the full contents of the file).

```
> GET /ubuntu/bionic/dists/bionic/InRelease HTTP/1.1
> Host: packages.fluentbit.io
> Range: bytes=4865-
> If-Range: Fri, 04 Oct 2019 00:00:00 GMT
>
< HTTP/1.1 206 Partial Content
< Server: Monkey/1.6.9
< Last-Modified: Fri, 11 Oct 2019 04:39:18 GMT
< Content-Length: 2
< Content-Range: bytes 4865-4866/4867
```

Here, the last modified timestamp of the file doesn't match the `If-Range:` value, but yet it returns a 206 response with the final 2 bytes of the file.

If the client detects this case by comparing the `Last-Modified:` value in the 206 response with what was requested, it could recover by falling back to making a second HTTP request for the full file.
However, apt-cacher-ng's HTTP client doesn't check that, and assumes that if the server returns a 206 response, it is behaving as expected (i.e. returning the last byte because the file has not been modified).
Therefore it appends all received data that's potentially > 1 byte in length to the file, starting with overwriting the last byte in the file with the first received byte, leading to the weird append behaviour.

### apt-cacher-ng workaround

Fortunately, there is a specific apt-cacher-ng option just to disable this: `VfileUseRangeOps`, which is described in `acng.conf` as:

> There some broken HTTP servers and proxy servers in the wild which don't
> support the If-Range header correctly and return incorrect data when the
> contents of a (volatile) file changed. Setting VfileUseRangeOps to zero
> disables Range-based requests while retrieving volatile files, using
> If-Modified-Since and requesting the complete file instead. Setting it to
> a negative value removes even If-Modified-Since headers.

### Server implementation

Why does the HTTP server at packages.fluentbit.io not fully support HTTP/1.1 range requests?
If we look at the Server header, it identifies itself as running version 1.6.9 of the [Monkey](http://monkey-project.com/) web server.

Unfortunately, that bit of HTTP seems to basically have never been implemented in Monkey.
If we look at the HTTP [request header enum](https://github.com/monkey/monkey/blob/v1.6.9/include/monkey/mk_http_parser.h#L92-L120) or the [request header parsing](https://github.com/monkey/monkey/blob/v1.6.9/mk_server/mk_http.c#L151-L157), the server understands the `Range:` (and `If-Modified-Since:`) header but not `If-Range:`.

### Server choice

It was quite puzzling why the Fluent Bit project would choose a relatively unknown HTTP server to host their downloads site.
The packages.fluentbit.io host is located on Rackspace's DFW network, so I initially thought it might be a [Rackspace Cloud Files](https://www.rackspace.com/cloud/files) thing, or a sponsor thing since Rackspace is listed on the project site as a sponsor.

But while browsing the GitHub repo further, I thought the Monkey server's author's username looked familiar from my recent work on implementing Fluentd and Fluent Bit -- [edsiper](https://github.com/edsiper) is _the_ creator and maintainer of Monkey and of Fluent Bit, looking at the [GitHub contributors page for Monkey](https://github.com/monkey/monkey/graphs/contributors) and [for Fluent Bit](https://github.com/fluent/fluent-bit/graphs/contributors).

The Monkey project seems pretty much dead, apart from whatever bits were library-ified and used in Fluent Bit, so I don't know if I'll bother filing an issue with it.
