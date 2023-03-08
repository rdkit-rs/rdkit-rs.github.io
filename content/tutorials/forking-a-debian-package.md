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

## Finding Updated Packages

It's great to be able to pull down the source code from the operating system's
repositories. But the development of that package must happen somewhere, new
package versions don't just snap in to existence, they flow from less stable
releases to more stable releases over time.

If we could just grab that latest work, take responsibility for its contents, we
could build it ourselves and hit fast-forward on versions.

Unfortunately, at this time of writing, I'm not sure where Ubuntu collaborates
on packaging definitions. But we know that Debian packages migrate to Ubuntu
over time, and it's safe to assume Debian is the originator of the RDKit
packaging definition.

Enter [salsa.debian.org](https://salsa.debian.org), a gitlab installation and
modern home for collaborating on packages in a format that might be familiar to
software developers. Specifically we can look at the
[DebiChem Pure Blend](https://wiki.debian.org/Debichem) (a special interest
group focused on making "Debian a good platform for chemists in their day-to-day
work") and more specifically we can look at the
[DebiChem Salsa Organization](https://salsa.debian.org/debichem-team) and their
[RDkit project](https://salsa.debian.org/debichem-team/rdkit).

We're going to take the RDKit definition from Salsa and make it our own.

## Using git to build source packages

In our opening _bash_ example we were able to pull sources from our regular
repository and just build the software. But when package definitions are tracked
in git we have to use modified techniques. And luckily for us the tooling around
packages in git is quite mature and just the right kind of esoteric.

The RDKit salsa repository carries on the tradition of tracking snapshots of the
upstream source code -- the repository does _not_ track
[the official RDKit repository](https://github.com/rdkit/rdkit), instead RDKit
on Salsa unpacks static releases and tracks their contents as-is. It does not
carry any commit history from upstream.

Each official RDKit release is tracked with a corresponding tag. For example,
RDKit 202209.3 gets the tag `202209.3`. But in the master branch there is a mix
of RDKit C++ code and Debian packaging description in the debian folder. The
tree looks something like this:

    % tree .
    .
    ├── CMakeLists.txt
    ├── CTestConfig.cmake
    ├── CTestCustom.ctest.in
    ├── Code
    │   ├── CMakeLists.txt
    [[ SNIP ]]
    ├── debian
    │   ├── TODO
    │   ├── changelog
    │   ├── clean
    │   ├── control
    │   ├── control.in
    │   ├── copyright
    │   ├── gbp.conf
    │   ├── librdkit-dev.install
    │   ├── librdkit1.install
    │   ├── librdkit1.lintian-overrides
    │   ├── patches
    │   │   ├── NoDownloads.patch
    [[ SNIP ]]
    │   ├── pgversions
    │   ├── python-rdkit.links
    │   ├── rdkit-data.install
    │   ├── rdkit-doc.dirs
    │   ├── rdkit-doc.doc-base
    │   ├── rules
    │   ├── salsa-ci.yml
    │   ├── source
    │   │   ├── format
    │   │   └── lintian-overrides
    │   ├── tests
    │   │   ├── control
    │   │   └── installcheck
    │   ├── upstream
    │   │   └── metadata
    │   └── watch

Updating to a newer RDKit release would essential mean clobbering everything
outside the `debian/` directory with the contents of the Release_YYYY_MM_B.zip
downloaded from the RDKit GitHub releases. It does seem messy to throw away the
commit history of the repository but Debian is only interest in packaging
defined releases, something considered stable software by the source authors.

We can use `gbp` (aka, git-buildpackage) to take a tag, build a source
`.orig.tar.xz` and then apply the `debian/` rules to construct a binary package.
Think of it like a git-centric workflow using the `dpkg-buildpackage` from
before alongside a variety of quality-improving tools we'll cover as we go.

But let's grab rdkit from salsa:

    # apt-get install -y git-buildpackage
    # git clone https://salsa.debian.org/debichem-team/rdkit.git
    # cd rdkit/

First we'll consider the top entry in the debian/changelog, this will dictate
the version assigned to the package:

    # cat debian/changelog
    rdkit (202209.3-2) UNRELEASED; urgency=medium


    -- Debichem Team <debichem-devel@lists.alioth.debian.org>  Sat, 14 Jan 2023 13:33:42 +0100

So the changelog doesn't say much for humans, but it gives the version
(`202209.3-2`), author (`Debichem Team`) and the date the changelog was
generated (`Sat, 14 Jan 2023 13:33:42 +0100`). So we expect if we use the right
incantations we can build rdkit 202209.3-2.

    rdkit # gbp buildpackage
    gbp:error: Pristine-tar couldn't verify "rdkit_202209.3.orig.tar.xz": fatal: path 'rdkit_202209.3.orig.tar.xz.delta' does not exist in 'refs/remotes/origin/pristine-tar'
    pristine-tar: git show refs/remotes/origin/pristine-tar:rdkit_202209.3.orig.tar.xz.delta failed

I don't know why we need a pristine tar ref. So we can tell gbp to skip it and
try again:

    rdkit# gbp buildpackage --git-no-pristine-tar
    gbp:info: Performing the build
    debuild: warning:     debian/changelog(l4): found trailer where expected start of change data
    LINE:  -- Debichem Team <debichem-devel@lists.alioth.debian.org>  Sat, 14 Jan 2023 13:33:42 +0100
    dpkg-buildpackage -us -uc -ui -i -I
    dpkg-buildpackage: warning:     debian/changelog(l4): found trailer where expected start of change data
    [[[ SNIP ]]]
    dpkg-buildpackage -us -uc -ui -i -I failed
    gbp:error: 'debuild -i -I' failed: it exited with 29

Ah, our old friend `dpkg-buildpackage` shows up again. But it fails, it looks
like the debian changelog is not suitable for a build just yet (submitting
broken code happens!), we can edit the changelog:

    rdkit# cat debian/changelog | head -n 10
    rdkit (202209.3-2) UNRELEASED; urgency=medium

    * Just filling this in

    -- Debichem Team <debichem-devel@lists.alioth.debian.org>  Sat, 14 Jan 2023 13:33:42 +0100

And try again:

    rdkit # gbp buildpackage --git-no-pristine-tar
    gbp:error: You have uncommitted changes in your source tree:
    gbp:error: On branch master
    Your branch is up to date with 'origin/master'.

Turns out the process of running buildpackage had applied patches and dirtied
our git checkout, making gbp suspicious the project was no longer suitable for
release. We can add on another flag to ignore dirty changes in the git repo:

    rdkit # gbp buildpackage --git-no-pristine-tar --git-ignore-new
    [[[ SNIP ]]]
    dpkg-checkbuilddeps: error: Unmet build dependencies: catch2 cmake dh-python doxygen flex imagemagick latexmk libboost-dev libboost-iostreams-dev libboost-python-dev libboost-regex-dev libboost-system-dev libboost-thread-dev libcairo-dev libcoordgen-dev libeigen3-dev libfreetype6-dev libmaeparser-dev librsvg2-bin libsqlite3-dev pandoc postgresql-server-dev-all python3-dev python3-numpy python3-pandas python3-pil | python3-imaging python3-recommonmark python3-sphinx python3-sqlalchemy rapidjson-dev texlive-latex-extra texlive-latex-recommended
    dpkg-buildpackage: warning: build dependencies/conflicts unsatisfied; aborting

And again we have an error, similar to the bash build before we are missing
dependencies required to compile the code. The tools won't automatically install
the build dependencies as that could be messy in a dev environment. Why don't we
just jump to sandboxing a build, no need in potentially muddying up a developers
OS.

## Sandboxing A Build

We want to build our software but we don't want to muddy up the base OS.
Fortunately `debootstrap` can populate a _fresh_ filesystem inside a directory,
a perfect OS for using `chroot` and friends to "move into" a sandbox. The gbp
tool prefers to use cowbuilder for moving into a sandbox, making it easy to
track changes done while building and then resetting the changes, avoiding a
potentially expensive debootstrap to recreate the filesystem.

    rdkit# gbp buildpackage --git-no-pristine-tar --git-ignore-new --git-pbuilder
    gbp:info: Building with (cowbuilder) for sid
    gbp:info: Performing the build
    Base directory /var/cache/pbuilder/base.cow does not exist
    gbp:error: 'git-pbuilder' failed: it exited with 1

We have to bootstrap that chroot jail ourselves, setting the distribution name
(by default it wants debian sid). And keep in mind this will inherit the mirror
used in the host OS, I ran this command from a jammy host on aarch64 hence the
use ports.ubuntu.com. If you are going to use a distribution other than Jammy
you need to specify `--mirror`.

    rdkit # DIST=jammy git-pbuilder create
    I: Invoking pbuilder
    I: forking: pbuilder create --buildplace /var/cache/pbuilder/base-jammy.cow --mirror http://ports.ubuntu.com/ubuntu-ports/ --distribution jammy --no-targz --extrapackages cowdancer
    W: /root/.pbuilderrc does not exist
    I: Running in no-targz mode
    W: cgroups are not available on the host, not using them.
    I: Distribution is jammy.
    I: Current time: Wed Mar  8 05:28:28 GMT 2023
    I: pbuilder-time-stamp: 1678253308
    I: Building the build environment
    I: running debootstrap
    /usr/sbin/debootstrap
    [[[ SNIP ]]]
    I: Resolving dependencies of required packages...
    I: Resolving dependencies of base packages...
    I: Checking component main on http://ports.ubuntu.com/ubuntu-ports...
    I: Retrieving adduser 3.118ubuntu5
    I: Validating adduser 3.118ubuntu5
    I: Retrieving apt 2.4.5
    I: Validating apt 2.4.5
    [[[ SNIP ]]]
    rdkit # du -hs /var/cache/pbuilder/base-jammy.cow
    422M    /var/cache/pbuilder/base-jammy.cow

and again we retry the sandboxed build, making sure to set the dist option so
gbp will reuse the deboostrap'd filesystem:

    rdkit# gbp buildpackage --git-no-pristine-tar --git-ignore-new --git-pbuilder --git-dist=jammy
    gbp:info: Building with (cowbuilder) for jammy
    gbp:info: Performing the build
    Building with cowbuilder for distribution jammy
    W: /root/.pbuilderrc does not exist
    I: using cowbuilder as pbuilder
    dpkg-checkbuilddeps: error: Unmet build dependencies: bison catch2 cmake dh-python doxygen flex imagemagick latexmk libboost-dev libboost-iostreams-dev libboost-python-dev libboost-regex-dev libboost-system-dev libboost-thread-dev libcairo-dev libcoordgen-dev libeigen3-dev libfreetype6-dev libmaeparser-dev librsvg2-bin libsqlite3-dev pandoc postgresql-server-dev-all python3-dev python3-numpy python3-pandas python3-pil | python3-imaging python3-recommonmark python3-sphinx python3-sqlalchemy rapidjson-dev tex-gyre texlive-fonts-recommended texlive-latex-base texlive-latex-extra texlive-latex-recommended
    W: Unmet build-dependency in source
    dh /usr/share/postgresql-common/pgxs_debian_control.mk --with python3 --buildsystem=cmake --parallel
    dh: error: unable to load addon python3: Can't locate Debian/Debhelper/Sequence/python3.pm in @INC (you may need to install the Debian::Debhelper::Sequence::python3 module) (@INC contains: /etc/perl /usr/local/lib/aarch64-linux-gnu/perl/5.34.0 /usr/local/share/perl/5.34.0 /usr/lib/aarch64-linux-gnu/perl5/5.34 /usr/share/perl5 /usr/lib/aarch64-linux-gnu/perl-base /usr/lib/aarch64-linux-gnu/perl/5.34 /usr/share/perl/5.34 /usr/local/lib/site_perl) at (eval 13) line 1.
    BEGIN failed--compilation aborted at (eval 13) line 1.

You might be tempted to try and install the build dependencies because you
google around and learn you can log in to the chroot environment and execute
arbitrary commands. Don't go down that path, the error your are seeing is not
really an error and will be resolved by the dpkg machinery. The issue is really
that `dh-python` is not installed on the host OS.

    rdkit# apt-get install -y dh-python
    [[[ SNIP ]]]

And then try again:

    rdkit# gbp buildpackage --git-no-pristine-tar --git-ignore-new --git-pbuilder --git-dist=jammy
    gbp:info: Building with (cowbuilder) for jammy
    gbp:info: Performing the build
    Building with cowbuilder for distribution jammy
    W: /root/.pbuilderrc does not exist
    I: using cowbuilder as pbuilder
    dpkg-checkbuilddeps: error: Unmet build dependencies: bison catch2 cmake doxygen flex imagemagick latexmk libboost-dev libboost-iostreams-dev libboost-python-dev libboost-regex-dev libboost-system-dev libboost-thread-dev libcairo-dev libcoordgen-dev libeigen3-dev libfreetype6-dev libmaeparser-dev librsvg2-bin libsqlite3-dev pandoc postgresql-server-dev-all python3-dev python3-numpy python3-pandas python3-pil | python3-imaging python3-recommonmark python3-sphinx python3-sqlalchemy rapidjson-dev tex-gyre texlive-fonts-recommended texlive-latex-base texlive-latex-extra texlive-latex-recommended
    W: Unmet build-dependency in source
    dh /usr/share/postgresql-common/pgxs_debian_control.mk --with python3 --buildsystem=cmake --parallel
    dh: error: Unknown sequence /usr/share/postgresql-common/pgxs_debian_control.mk (choose from: binary binary-arch binary-indep build build-arch build-indep clean install install-arch install-indep)
    debian/rules:37: /usr/share/postgresql-common/pgxs_debian_control.mk: No such file or directory
    make: *** [debian/rules:40: /usr/share/postgresql-common/pgxs_debian_control.mk] Error 25
    gbp:error: 'git-pbuilder' failed: it exited with 2

Now our `debian/rules` (aka, a Debian packaging-flavored Makefile) is relying on
the existence of a file but failing to load it, causing the Makefile to fail
before package resolution can happen. This is a confusing situation and
definitely points to tool roughness.

Let's use `apt-file` to see what packages provides the missing path:

# apt-get install -y apt-file

# apt-file update

# apt-file search /usr/share/postgresql-common/pgxs_debian_control.mk

postgresql-server-dev-all: /usr/share/postgresql-common/pgxs_debian_control.mk

And we definitely had `postgresql-server-dev-all` in the build-depends of
`debian/control`. Sounds like a race, the `debian/rules` can't evaluate with a
package from build-depends and the build-depends can't be satisfied until
`debian/rules` is run. How about we shoe-horn this package in to the
debootstrap'd filesystem?

From the host OS let's see if we can understand this package more:

    # apt-cache show postgresql-server-dev-all
    # heavily edited output
    Package: postgresql-server-dev-all
    Section: universe/database
    Origin: Ubuntu
    Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
    Original-Maintainer: Debian PostgreSQL Maintainers <team+postgresql@tracker.debian.org>
    Filename: pool/universe/p/postgresql-common/postgresql-server-dev-all_238_arm64.deb

I see a lot of references to `universe` but I only saw `main` in the
`debootstrap` invocation, which I copy here:

    I: forking: pbuilder create --debootstrapopts --include="postgresql-server-dev-all" --buildplace /var/cache/pbuilder/base-jammy.cow --mirror http://ports.ubuntu.com/ubuntu-ports/ --distribution jammy --no-targz --extrapackages cowdancer
    I: Checking component main on http://ports.ubuntu.com/ubuntu-ports...

but no `universe`. Those repositories are described here:
https://help.ubuntu.com/community/Repositories/Ubuntu. But essentially
`universe` gets less attention from Ubuntu/Canonical but it has this postgresql
dev package.

so let's destroy our incomplete filesystem and bootstrap it again, including
`universe` in the package list:

    # rm -rf /var/cache/pbuilder/base-jammy.cow
    # DIST=jammy git-pbuilder create --debootstrapopts=--components="main,universe"

we are not installing the missing package, hoping instead the machinery for
building a package will find it and install it

## Further reading

- http://honk.sigxcpu.org/projects/git-buildpackage/manual-html/gbp.html
- https://en.wikipedia.org/wiki/Chroot
