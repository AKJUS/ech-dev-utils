# Ancillary ECH developer content

This is a [DEfO](https://defo.ie) project production.

The current development branch of our Encrypted ClientHello (ECH) enabled fork
of OpenSSL is
[ECH-draft-13c](https://github.com/sftcd/openssl/tree/ECH-draft-13c), under the
github ``sftcd`` account, but you can use the one
[here](https://github.com/defo-project/openssl), which is less likely to
undergo changes. That ``sftcd`` branch also contains some of this material, but
now that we've submitted a [PR for upstream
OpenSSL](https://github.com/openssl/openssl/pull/22938), this material needs
somewhere else to live in the longer term, so we've moved it here.

The content here includes scripts for doing ECH things, sample configurations
and HOWTOs for building and testing ECH-enabled things.

We also include new CI workflows and that do a merge with upstream, build and
run basic tests of the various packages here, to check for whenever we get a
mismatch between upstream and our ECH-enabled forks. Those CI jobs are in the
``.github/workflows/packages.yaml`` of each repo, and are further described
[here](howtos/CI-builds.md).

These scripts and howtos have been tested in an Ubuntu 23.04 development
environment and as a default assume that you have other code repos installed
and built in e.g.  ``$HOME/code/openssl`` or ``$HOME/code/nginx`` etc.

## ECH-style wrappers for OpenSSL command line tools (and related)

- [localhost-tests.md](howtos/localhost-tests.md) is a HOWTO for using the
  scripts below
- [echcli.sh](scripts/echcli.sh) is a relatively comprehensive wrapper for
  ``openssl s_client`` that allows one to play with lots of ECH options
- [echsvr.sh](scripts/echsvr.sh) is a relatively comprehensive wrapper for
  ``openssl s_server`` that allows one to play with lots of ECH options
- [make-example-ca.sh](./scripts/make-example-ca.sh) creates fake x.509 certs
  for example.com and the likes of foo.example.com so we can use the scripts
and configs here for localhost tests - you have to have gotten that to work
before ``echsvr.sh`` can be used for localhost tests

## Pure test scripts

Once you have an OpenSSL build in ``$HOME/code/openssl`` you can just
run these. Note these are deliberately sedate, but that's ok.

- [agiletest.sh](scripts/agiletest.sh) tests ECH using ``openssl s_client`` and
  ``openssl s_server`` with the various algorithm combinations that are
  supported for ECHConfig values - this isn't used so much any more as
  the ``make test`` target in the OpenSSL build now does the equivalent
  and is much quicker

## Scripts to play with ECHConfig values (that may get put in the DNS)

We defined a new PEM file format for ECH key pairs, specified in
[draft-farrell-tls-pemesni/](https://datatracker.ietf.org/doc/draft-farrell-tls-pemesni/).
(That's an individual Internet-draft and doesn't currently have any standing in
terms of IETF process, but it works and our code uses it.) Some of the scripts
below depend on that.

- [mergepems.sh](scripts/mergepems.sh) merges the ECHConfigList values from two
  ECH PEM files
- [pem2rr.sh](scripts/pem2rr.sh) encodes the ECHConfigList from an ECH PEM file
  into a validly (ascii-hex) encoded HTTPS resource record value
- [splitechconfiglist.sh](scripts/splitechconfiglist.sh) splits the
  ECHConfigList found in a PEM file into constituent parts (if that has more
than one ECHConfig) - the output for each is a base64 encoded ECHConfigList
with one ECHConfig entry (i.e., one public name/key)
- [makecatexts.sh](scripts/makecatexts.sh) allows one to create a file with a
  set of cat pictures suited for use as the set of extensions for an
ECHConfigList that can be added to a PEM file via the ``openssl ech`` command -
those need to be *very* small cat pictures though if you want the resulting
HTTPS RR to be usable in the DNS - note that as nobody yet has any real use for
ECHConfig extensions (and they're a bad idea anyway;-) this is really just used
to try, but hopefully fail, to break things

## Client HOWTOs

- the HOWTO for
  [curl](https://github.com/sftcd/curl/blob/ECH-experimental/docs/ECH.md) is no
  longer in this repo but is now part of a curl PR branch
- we have a [HOWTO](howtos/cpython.md) for building and testing our CPython fork
- [wget.md](howtos/wget.md) has notes about how ECH-enabling wget is
  non-trivial
- the HOWTO for testing versus the Boringssl is [here](howtos/boring.md)

## Web server build HOWTOs, configs and test scripts

The HOWTO files here have build instructions mostly, but also some notes about
code changes. The config files are minimal files for a localhost test with the
relevant server, but of course include our new ECH config stanzas. The scripts
are used to run the server-side of localhost tests, generally the HOWTO has a
way to use ``echcli.sh`` for the client side.

(For the pedantic amongst you, yes haproxy isn't really a web server but
just a super-fast proxy, but... meh:-)

### Server configs preface - key rotation and slightly different file names

Most of the server configs below allow one to name a directory from which all
ECH PEM files will be loaded into the server. That allows for sensible ECH key
rotation, e.g., to publish the most recent one in DNS, but to also allow some
previously, but no longer, published ECH key(s) to still be usable, by clients
with stale DNS caches or where some DNS TTL fun has been experienced. You can
then reload the server config without having to change the config file which
seems like a reasonably good thing.

The model we use in test servers is to publish new ECH keys hourly, but for the
most recent three keys to still be usable for ECH decryption in the server.
That's done via a cron job that handles all the key rotation stuff with a bit
of orchestration between the key generator, ECH-enabled TLS server and DNS
"zone factory." (If you're interested in details, there's a [TLS working group
draft](https://datatracker.ietf.org/doc/html/draft-ietf-tls-wkech) that
describes a way to do all that, and a [bash script
implementation](https://github.com/sftcd/wkesni/blob/master/wkech-04.sh) that's
what we use as the cron job for our test servers.)

The upshot of all that is that servers want to load all the ECH PEM files from
a named directory, but it's also possible that other PEM files may exist in the
same place that contain x.509 rather than ECH keys, so to avoid problems, the
servers will attempt to load/parse all files from the named directory that are
called ``*.ech`` rather than the more obvious ``*.pem``.

For these configs and test scripts then, and assuming you've already gotten the
[localhost test](howtos/localhost-tests.md) described above working and are
using the same directory you setup before, (with the fake x.50 CA ``cadir``
etc.), you should do the following (or similar) before trying to run the
various server-specific tests:

```bash
    cd $HOME/lt
    mkdir echkeydir
    cp echconfig.pem echkeydir/echconfig.pem.ech
```

That's a bit convoluted, sorry;-) We're also not entirely sure it's done fully
consistently for all servers, but if not, we'll fix it.

### Server details

For each of the servers below, read the HOWTO then you can play with the test
script etc. The test scripts basically each start a server, then connect to the
relevant server port using ``echcli.sh`` in different ways, including testing
connecting to the ``public_name``, GREASEing ECH, a nominal use of ECH and
triggering ECH+HRR. There are utility bash functions in
[funcs.sh](./scripts/funcs.sh).

- Nginx
    - HOWTO: [nginx.md](howtos/nginx.md)
    - config template: [nginxmin.conf](configs/nginxmin.conf)
    - test script: [testnginx.sh](scripts/testnginx.sh)
- Apache
    - HOWTO: [apache2.md](howtos/apache2.md)
    - config: [apachemin.conf](configs/apachemin.conf)
    - test script: [testapache.sh](scripts/testapache.sh)
    - to run apache in gdb: [apachegdb.sh](scripts/apachegdb.sh)
- Lighttpd
    - HOWTO: [lighttpd.md](howtos/lighttpd.md)
    - config: [lighttpdmin.conf](configs/lighttpdmin.conf)
    - test script: [testlighttpd.sh](scripts/testlighttpd.sh)
- Haproxy (shared-mode):
    - HOWTO: [haproxy.md](howtos/haproxy.md)
    - frontend haproxy config: [haproxymin.conf](configs/haproxymin.conf)
    - backend lighttpd config: [lighttpd4haproxymin.conf](configs/lighttpd4haproxymin.conf)
    - test script: [testhaproxy.md](scripts/testhaproxy.sh)
- ECH split-mode with nginx or haproxy frontend
    - HOWTO: [split-mode.md](howtos/split-mode.md)
    - frontend nginx config: [nginxsplit.conf](configs/nginxsplit.conf)
    - frontend haproxy config: [haproxymin.conf](configs/haproxymin.conf)
    - backend lighttpd config: [lighttpdsplit.conf](configs/lighttpdsplit.conf)
    - test script: [testsplitmode.sh](scripts/testsplitmode.sh)

## Misc. files

- [cat.ext](misc/cat.ext) is a cat picture encoded as an ECHConfig extension suited
  for inclusion in an ECHConfigList - for the cat lovers out there, the
  original image is [cat.jpg](misc/cat.jpg), downsized to [scat.png](misc/scat.png)
- [extsfile](misc/extsfile) contains two ECHConfig extensions (suited for inclusion
  in an ECHConfigList), the first being of zero length, the second being a very
  small picture (I forget of what;-)

## Misc. scripts

A few bits'n'pieces that've been useful along the way, but mostly haven't been
used in some time:

- [nssdoech.sh](scripts/nssdoech.sh) tests ECH using an NSS client build (mostly helped
  with interop)
- [bssl-oss-test.sh](scripts/bssl-oss-test.sh) tests ECH using a boringssl build
  (mostly helped with interop)
- [dnsname.sh](scripts/dnsname.sh) decodes a DNS wire format encoding of a DNS name
  (just a useful snippet, not so much for using in anger)
- [scanem.sh](scripts/scanem.sh) compares two OpenSSL source trees and reports on which
  files differ
- [runindented.sh](scripts/runindented.sh) is a bash function for indenting things (not
  currently used, but was, and may be again sometime)
- [localhost-tests.sh](scripts/localhost-tests.sh) runs a couple of the localhost
  tests, as used (for now) in package testing - may well be enhanced soon.
- [test-cases](./test-cases) has scripts to generate test cases for ECH
  used by [test-cases-gen.py](scripts/test-cases-gen..py).
- [ech-check.php](scripts/ech-check.php) is a version of the PHP code at
  [defo.ie/ech-check.php](https://defo.ie/ech-check.php)
- [domainechprobe.php](echprobe/domainechprobe.php) is a version of the
  PHP code at [https://test.defo.ie/domainechprobe.php](https://test.defo.ie/domainechprobe.php)
- [selenium_test.py](scripts/selenium_test.py) is our selenium test
- [smoke_ech_curl.sh](scripts/smoke_ech_curl.sh) runs through a list of sites known
  to support ECH and reports on status
- [ech_url.go](scripts/ech_url.go) is a golang program to try use ECH when accessing
   a URL
- [smoke_ech_golang.sh](scripts/smoke_ech_golang.sh) is a script that calls
  [ech_url.go](scripts/ech_url.go) for a set of test URLs and records results

