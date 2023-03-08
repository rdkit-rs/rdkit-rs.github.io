---
title: Forking a Debian Package
geekdocHidden: false
---

Debian is an operating system with a rich packaging format and dependency
resolution system. It runs on the Linux kernel and provides _over 80,000
packages_. Packages ranging from shells like _bash_, server software like
_nginx_, desktop software like _Firefox_, and even full-blown desktop
environments like _Gnome_. The Ubuntu operating system is a Debian variant which
adopts many of the same packages as Debian, providing a commercial facade and a
different, potentially favorable package lifecycle. Every two years Ubuntu names
a Long Term Support (LTS) release, in this article we'll focus on Ubuntu 22.04
codenamed Jammy, an LTS supported through 2027.

RDKit is a C++ library and suite of Java/Python modules. Unfortunately Ubuntu
Jammy is providing RDKit 202109 and will not update the major version. Ubuntu
releases pin their entire dependency tree except for bug and security patches.
The RDKit-rs project needs to be flexible and stay up to date on RDKit changes
upstream. We anticipate finding bugs and potentially customizing the RDKit C++
to meet our needs. In short, the RDKit-rs project needs a lot more control over
RDKit.

## How can we fork a Debian package?

Forking is a very common thing to solve in the world of git, you just hit the
`fork` button on Github and copy the repository to your account. Or perhaps you
need to fork a docker image? You can either layer your changes on top of the
original image or you can take the Dockerfile and customize it to your needs,
with easy distribution through `docker push`/`docker pull` to a repository.
Unfortunately things are not so easy in the Debian world.

We'll need to understand:

- what a Debian package is
- how to build a package
- how RDKit is packaged
- how to distribute custom Debian packages. We won't cover starting a package
  from scratch.

## Debian Source Packages

One goal of Debian is to run herd on the 80,000+ open source packages. Debian
does not maintain forks of these projects, instead preferring static snapshots
of the package source code. Source code is packaged as an untouched `.tar.gz`
and can be uploaded to Debian repositories for download by clients.

Once the source is packaged it's time to answer the questions "how do we build
something usable out of this source code?". No need to learn how to build every
piece of software, the mechanical steps are captured by the Debian packaging
rules. And similarly, no need to manually fetch build-time and run-time
dependencies for software, those can be tracked by the Debian package
description.

## Building A Binary Package From Source

A classic Debian packaging project laid out on disk might look like:

    # apt-get source bash
    # tree .
    |-- bash-5.1
    |   |-- ABOUT-NLS
    |   |-- AUTHORS
    |-- bash_5.1-6ubuntu1.debian.tar.xz
    |-- bash_5.1-6ubuntu1.dsc
    `-- bash_5.1.orig.tar.xz

The `bash-5.1` directory is unpacked for your convenience using the
`.orig.tar.xz`. You will need to be inside this directory to build the package.

The `bash_5.1-6ubuntu1.debian.tar.xz` contains all the Debian patches and build
instructions. This is a big source of value as it provides a common surface to
the package and once you learn how the parts work you can customize any package.

The `bash_5.1-6ubuntu1.dsc` contains the metadata like `build-depends` so you
can easily fetch required packages to build the software, or `depends` for
required packages to run the software. It also includes security features like
GPG signature from the packaging author.

And finally `bash_5.1.orig.tar.xz` is the untouched snapshot of the source code
from the upstream project. No need to go the bash website to fetch this
software, and no need to worry about the software project deleting a version of
software.

Build the software with the packaging rules as-is:

    # ls
    bash-5.1  bash_5.1-6ubuntu1.debian.tar.xz  bash_5.1-6ubuntu1.dsc  bash_5.1.orig.tar.xz
    # cd bash-5.1
    bash-5.1 # dpkg-buildpackage
    [[ SNIP ]]
    dpkg-checkbuilddeps: error: Unmet build dependencies: bison libncurses5-dev texinfo texi2html sharutils locales time texlive-latex-base ghostscript texlive-fonts-recommended man2html-base

The Debian packaging system doesn't need to build the software to know we don't
have all our prerequisites. We can just install those with the following:

    apt-get install -y bison libncurses5-dev texinfo texi2html sharutils locales time texlive-latex-base ghostscript texlive-fonts-recommended man2html-base

In the future we'll use mechanisms that auto install build dependencies. But
after running that install manually we try again:

    bash-5.1# dpkg-buildpackage
    dpkg-buildpackage: info: source package bash
    dpkg-buildpackage: info: source version 5.1-6ubuntu1
    dpkg-buildpackage: info: source distribution jammy
    dpkg-buildpackage: info: source changed by Matthias Klose <doko@ubuntu.com>
    dpkg-buildpackage: info: host architecture arm64
    [[[ SNIP ]]]
    Beginning configuration for bash-5.1-release for aarch64-unknown-linux-gnu

    checking for gcc... gcc
    checking whether the C compiler works... yes
    checking for C compiler default output file name... a.out
    checking for suffix of executables...
    checking whether we are cross compiling... no
    checking for suffix of object files... o
    checking whether we are using the GNU C compiler... yes
    checking whether gcc accepts -g... yes

What's great is that we don't know all the details for how bash is built. No
wrangling autotools, cmake, Makefile, etc. The goal of building a binary has
been boiled down to invoking `dpkg-buildpackage`. And for the most part you will
use pre-built binaries from the Debian or Ubuntu build farms.

## Mixing In An Updated Package

It's great to be able to pull down the source code from the operating system's
repositories. But the development of that package must happen somewhere, new
package versions don't just snap in to existence, they flow from less stable
releases to more stable releases over time.

If we could just grab that latest work, take responsibility for its contents, we
could build it ourselves and hit fast-forward on versions.

Unfortunately, at this time of writing, I'm not sure where Ubuntu collaborates
on packaging definitions. But we know that Debian packages migrate to Ubuntu
over time and it's safe to assume Debian is the originator of the RDKit
packaging definition.

Enter [salsa.debian.org](https://salsa.debian.org), a gitlab installation and
modern home for collaborating on packages in a format that might be familiar to
software developers. Specifically we can look at the
[DebiChem Pure Blend](https://wiki.debian.org/Debichem) (a special interest
group focused on making "Debian a good platform for chemists in their day-to-day
work") and more specifically we can look at the
[DebiChem Salsa Organization](https://salsa.debian.org/debichem-team) and their
[RDkit project](https://salsa.debian.org/debichem-team/rdkit).
