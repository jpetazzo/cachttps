# CAChe HTTPS

The year is 2025, and a lot of our build and deployment pipelines
rely on downloading things from github.com (whether it's cloning
repositories, downloading tarballs, binary releases, etc).

At some point, the IT industry collectively decided that HTTP
Was Bad, Actually and that everything should indiscriminately
go over HTTPS instead, because Security Is Important.

It turns out that gruffy neckbeards and their cat-ears wearing
colleagues had been using digital signatures for decades
and distributing packages with HTTP, FTP, rsync, CD-ROM,
avian carriers, and anything and everything invented by mankind
to carry data from point A to point B, and HTTPS didn't really
improve that situation, but it certainly made a bunch of
security auditors feel warm and fuzzy inside (at least I
suppose so; I've never been inside a security auditor so
I don't know what it feels like).

What HTTPS *did* certainly disrupt, however, is the ability
to transparently proxy and cache our downloads. You used to
be able to install squid, drop in a small config file, an
ipfwadm or ipchains rule, and *voilà*, all your Debian boxes
in your home, office or campus would automatically fetch
packages from a shared cache.

(If you have strong feelings about the Great Importance
of using HTTPS all day every day, skip to the end of
that text to read more about that.)


Also, for some reason that I'm not privy too, the network
team at GitHub seems to be unable to (or prevented from)
deploying IPv6, which means that it's very difficult to
deploy something on a n IPv6-only (single stack) machine.

Let's strike these two birds with one small NGINX config file.

## How this works

This is a proxy+cache that will fetch GitHub URLs for
you, serve them, and cache them very agressively.

To see it in action:
```bash
docker compose up -d
```

Then, you can download things from GitHub by accessing:
```bash
curl http://localhost:3131/https://github.com/jpetazzo.keys
curl http://localhost:3131/https://github.com/octocat/Spoon-Knife/archive/refs/heads/main.zip
```

Then, you can run it somewhere in your infra (either with
the provided Compose file, or by dropping the proxy.conf file
in an NGINX install deployed with your poison of choice),
and update your deployment scripts to prepend every GitHub
URL with http://<cachttps-address:3131>/. That's it.

## But... I thought HTTPS was more secure?

Yes and no. As always, the world is a nuanced place,
and reducing it to 140-character hot takes isn't helping.

Here are the main arguments in favor of using HTTPS
(instead of HTTP or other non-encrypted transports)
and why they actually don't hold water.

> HTTPS prevents bad people from hijacking your
> downloads, so it helps making sure that you
> don't download a malicious payload (because
> someone replaced a software update with a malware).

HTTPS does indeed secure the transport layer between
your distribution point (e.g. package mirror) and your
computer, but it doesn't secure it end-to-end (starting
with the person or organization who created the package).
This means that somebody with access to distribution
mirrors, or the CDN they use, could still hijack
the payload.

Furthermore, HTTPS doesn't provide trust, or rather,
it shifts the trust on the certificates that are
recognized by your machine. In other words, if a
root CA gets compromised, or if someone manages to
slip a fake CA on your machine, you're toast.

Instead, it's a good idea to *sign* software updates
and verify these signatures. This is something that
all major distros have been doing for a very long time.
Sometimes at a very granular level (verifying signatures
of individual packages), sometimes at a broader level
(verifying signature of a "releases" or index file).

> But, even if you use signatures, if you install your
> software upgrades with HTTP, it's easy for an attacker
> to selectively block the download of a selective package
> (to prevent patching a vulnerability, for instance).

This is true. With HTTP, we can easily block the download
of a single package, while with HTTPS, we would have to
block all downloads from the distribution's servers,
and if the distribution servers are a CDN, blocking the
whole CDN could be difficult - or at least, easy to
notice from the victim's perspective.

On the other hand, if we're using HTTP, it's easier
to have multiple mirrors and try them all. It then becomes
harder for an attacker to block that.

If the trust is established by signature ("I trust
this file because it bears a valid signature") instead
of by network origin ("I trust this file because I
downloaded it from download.mydistro.org"), we can
download the file from anywhere (multiple mirrors,
HTTP or HTTPS) and *then* verify the signature.
If, for instance, we try to download our updates
from a few HTTP servers, and fallback to HTTPS
when there are download errors, we keep the advantage
of HTTP (transparent caching) but without being
vulnerable to attacks, thanks to the HTTPS fallback.

> Verifying signatures is complicated. It's easier for
> me to trust my download because I obtained it from
> the right address.

This is absolutely true. We're lazy creatures and
we very often prefer to do the easy thing instead of
the right thing, and many of us (me included) will
happily download a tarball from GitHub and consider
it good thanks to the relative trust that we place
into HTTPS.

