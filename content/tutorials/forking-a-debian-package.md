---
title: Forking a Debian Package
geekdocHidden: false
---

Debian is an operating system with a rich packaging format and dependency
resolution system. It runs on the Linux kernel and provides _over 80,000
packages_, including shells like _bash_, server software like _nginx_, desktop
software like _Firefox_, and even full-blown desktop environments like _Gnome_.
The Ubuntu operating system is a Debian variant that adopts many of the same
packages as Debian, while also offering commercial support and a potentially
more favorable package lifecycle. Every two years, Ubuntu names a Long Term
Support (LTS) release, and in this article, we'll focus on Ubuntu 22.04,
codenamed Jammy, which is an LTS release supported through 2027.

RDKit is a C++ library and suite of Java/Python modules. Unfortunately Ubuntu
Jammy is providing RDKit 202109 and will not update the major version. Ubuntu
releases pin their entire dependency tree except for bug and security patches.
The RDKit-rs project needs to be flexible and stay up to date on RDKit changes
upstream. We anticipate finding bugs and potentially customizing the RDKit C++
to meet our needs. In short, the RDKit-rs project needs a lot more control over
RDKit.

## How can we fork a Debian package?

Forking is common in the world of git, you just hit the `fork` button on Github
and copy the repository to your account. Perhaps you need to fork a docker
image? You can either layer your changes on top of the original image or you can
take the Dockerfile and customize it to your needs, with easy distribution
through `docker push`/`docker pull` to a repository. Unfortunately things are
not so easy in the Debian world -- or at least they're not documented in a way I
found easy to follow. This tutorial serves as my own notes but should empower
others to fork high quality Debian packages and remix them for their own
purposes.

In order to fork effectively we'll need to understand:

- what a Debian package is
- how to build a package
- how RDKit is packaged
- how to distribute custom Debian packages. We won't cover starting a package
  from scratch.

## Debian Source Packages

One goal of Debian is to run herd on the
[60,000+ open source packages](https://debian.pkgs.org/) . Debian does not
maintain forks of these projects, instead preferring static snapshots of the
package source code. Source code is packaged as an untouched `.tar.gz` and can
be uploaded to Debian repositories for download by clients.

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

    rdkit# rm -rf /var/cache/pbuilder/base-jammy.cow
    rdkit# DIST=jammy git-pbuilder create --debootstrapopts=--components="main,universe"
    [[[ SNIP ]]]
    rdkit# gbp buildpackage --git-no-pristine-tar --git-ignore-new --git-pbuilder --git-dist=jammy
    dh /usr/share/postgresql-common/pgxs_debian_control.mk --with python3 --buildsystem=cmake --parallel
    make: dh: No such file or directory
    debian/rules:37: /usr/share/postgresql-common/pgxs_debian_control.mk: No such file or directory
    make: *** [debian/rules:40: /usr/share/postgresql-common/pgxs_debian_control.mk] Error 127
    gbp:error: 'git-pbuilder' failed: it exited with 2

still erroring on the missing file, the build machinery is not loading the
dependency, so let's recreate the filesystem again but force the install of
`postgresql-server-dev-all` and `debhelper` (the source of `/usr/bin/dh`) and
dh-python (the source of `python3` support in builds):

    # rm -rf /var/cache/pbuilder/base-jammy.cow
    # DIST=jammy git-pbuilder create --debootstrapopts=--components="main,universe" --debootstrapopts=--include="postgresql-server-dev-all debhelper dh-python"

at this point I have used `DIST=jammy git-pbuilder login` to hop in to the
filesystem and confirm files exist and things are still not working. I'm
suspicious if this part of the build is inside the chroot jail or on the host,
let's install on the host:

    # apt-get install -y postgresql-server-dev-all debhelper dh-python
    # gbp buildpackage --git-no-pristine-tar --git-ignore-new --git-pbuilder --git-dist=jammy
    [[[ SNIP ]]]
    pg_buildext checkcontrol
    --- debian/control      2023-03-08 17:17:49.451838004 +0000
    +++ debian/control.EuTFyl       2023-03-08 17:51:28.825813007 +0000
    @@ -166,10 +166,10 @@
    .
    This package contains the header files.

    -Package: postgresql-15-rdkit
    +Package: postgresql-14-rdkit
    Architecture: any
    Section: database
    -Depends: ${misc:Depends}, ${shlibs:Depends}, postgresql-15
    +Depends: ${misc:Depends}, ${shlibs:Depends}, postgresql-14
    Suggests:
    Description: Cheminformatics and machine-learning software (PostgreSQL Cartridge)
    RDKit is a Python/C++ based cheminformatics and machine-learning software
    Error: debian/control needs updating from debian/control.in. Run 'pg_buildext updatecontrol'.
    If you are seeing this message in a buildd log, a sourceful upload is required.
    make: *** [/usr/share/postgresql-common/pgxs_debian_control.mk:9: debian/control] Error 1
    gbp:error: 'git-pbuilder' failed: it exited with 2

finally a new error, pointing to a version mismatch between Debian (postgres 15)
and Ubuntu Jammy (postgres 14), let's try running the recommended fix
`pg_buildext updatecontrol`:

    rdkit# pg_buildext updatecontrol

now the `debian/control.in` has been re-rendered to update `debian/control` and
we can continue the build:

    # gbp buildpackage --git-no-pristine-tar --git-ignore-new --git-pbuilder --git-dist=jammy
    gbp:info: Building with (cowbuilder) for jammy
    gbp:info: Performing the build
    [[[ SNIP ]]]
    debian/rules override_dh_auto_clean
    make[1]: Entering directory '/rdkit'
    find /rdkit -name "*.pyc" | xargs rm -f
    [[[ SNIP ]]]
    Preparing to unpack .../075-libxcb-shm0_1.14-3ubuntu3_arm64.deb ...
    Unpacking libxcb-shm0:arm64 (1.14-3ubuntu3) ...
    [[[ SNIP ]]]
    make[1]: Entering directory '/build/rdkit-202209.3'
    dh_auto_configure -- -DCMAKE_BUILD_TYPE=None -DCMAKE_SKIP_RPATH=ON -DRDK_INSTALL_INTREE=OFF -DRDK_INSTALL_STATIC_LIBS=OFF -DRDK_BUILD_THREADSAFE_SSS=ON -DRDK_BUILD_PYTHON_WRAPPERS=ON -DRDK_OPTIMIZE_POPCNT=OFF -DRDK_USE_URF=OFF -DRDK_INSTALL_COMIC_FONTS=OFF -DRDK_BUILD_CAIRO_SUPPORT=ON -DBoost_NO_BOOST_CMAKE=TRUE -DCMAKE_INSTALL_PREFIX=/usr -DCATCH_DIR=/usr/include/catch2 -DPYTHON_EXECUTABLE=/usr/bin/python3.10 ../
    [[[ SNIP ]]]
    [  0%] Building CXX object Code/RDGeneral/CMakeFiles/RDGeneral.dir/Invariant.cpp.o
    cd /build/rdkit-202209.3/obj-aarch64-linux-gnu/Code/RDGeneral && /usr/bin/c++ -DBOOST_SERIALIZATION_DYN_LINK -DRDGeneral_EXPORTS -DRDKIT_DYN_LINK -DRDKIT_RDGENERAL_BUILD -DRDK_64BIT_BUILD -DRDK_BUILD_COORDGEN_SUPPORT -DRDK_BUILD_DESCRIPTORS3D -DRDK_BUILD_MAEPARSER_SUPPORT -DRDK_BUILD_THREADSAFE_SSS -DRDK_HAS_EIGEN3 -DRDK_TEST_MULTITHREADED -DRDK_USE_BOOST_SERIALIZATION -DRDK_USE_STRICT_ROTOR_DEFINITION -I/build/rdkit-202209.3/External -I/usr/include/python3.10 -I/usr/lib/python3/dist-packages/numpy/core/include -I/build/rdkit-202209.3/Code -isystem /usr/include/eigen3 -g -O2 -ffile-prefix-map=/build/rdkit-202209.3=. -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2 -Wdate-time -D_FORTIFY_SOURCE=2 -Wno-deprecated -Wno-unused-function -fno-strict-aliasing -Wall -Wextra -fPIC -fPIC -std=gnu++17 -MD -MT Code/RDGeneral/CMakeFiles/RDGeneral.dir/Invariant.cpp.o -MF CMakeFiles/RDGeneral.dir/Invariant.cpp.o.d -o CMakeFiles/RDGeneral.dir/Invariant.cpp.o -c /build/rdkit-202209.3/Code/RDGeneral/Invariant.cpp

Finally! We see the C++ compiler is being used, the rdkit source code is
compiled. At the end of the process we will have a collection of debian package
files suitable for installing on Jammy.

Takeaways:

- bootstrapping the cowbuilder filesystem needs main and universe
- the build host stills needs a fair bit of configuration, installing
  `postgresql-server-dev-all`, `debhelper`, `dh-python` which are used outside
  the cowbuilder context
- you have to get used to running commands, reading opaque errors, deleting
  resources and trying again
  - kudos to cowbuilder for aggressively caching apt downloads

## Moving the process to GitHub Actions

In general, we never want to ship production assets from a workstation
environment. We want to build our artifacts in an auditable, open environment
with logging and timestamps. Enter GitHub Actions, a git event-triggered runtime
environment where we can recreate the above workflow.

You will need to create a `.github/workflows/ci.yml` with all the above steps
redone in a clean manner:

    name: Build and publish Debian packages

    on:
      push: {}

    jobs:
      build_amd64:
        runs-on: buildjet-16vcpu-ubuntu-2204
        env:
          DIST: jammy
          ARCH: amd64
          DEB_BUILD_OPTIONS: nocheck

        steps:
          - name: Check out the repo
            uses: actions/checkout@v3
            with:
              fetch-depth: 0

          - name: Install build tools
            run: sudo apt-get update; sudo apt-get install -y git build-essential git-buildpackage debhelper dh-python

          - name: Create base image
            run: git-pbuilder create

          - name: Remove github stuff
            run: rm -rf .github/

          - name: Build package
            run: gbp buildpackage --git-ignore-new --git-pbuilder --git-no-pristine-tar --git-arch=$ARCH --git-dist=$DIST -us -uc

          - name: Check for debs
            run: ls -alh .; ls -alh ..

and at the end of the run you will have `.deb` files in the filesystem of the
github actions runner:

    total 245M
    drwxrwxr-x  3 runner runner 4.0K Mar  7 19:40 .
    drwxrwxr-x  6 runner runner 4.0K Mar  7 19:30 ..
    -rw-r--r--  1 runner runner 380K Mar  7 19:39 librdkit-dev_202209.3-2_amd64.deb
    -rw-r--r--  1 runner runner  88M Mar  7 19:39 librdkit1-dbgsym_202209.3-2_amd64.ddeb
    -rw-r--r--  1 runner runner 4.3M Mar  7 19:39 librdkit1_202209.3-2_amd64.deb
    -rw-r--r--  1 runner runner 107K Mar  7 19:39 postgresql-15-rdkit_202209.3-2_amd64.deb
    -rw-r--r--  1 runner runner  50M Mar  7 19:39 python3-rdkit-dbgsym_202209.3-2_amd64.ddeb
    -rw-r--r--  1 runner runner 4.2M Mar  7 19:39 python3-rdkit_202209.3-2_amd64.deb
    -rw-r--r--  1 runner runner  13M Mar  7 19:40 rdkit-data_202209.3-2_all.deb
    drwxrwxr-x 16 runner runner 4.0K Mar  7 19:32 rdkit-debian
    -rw-r--r--  1 runner runner 7.4M Mar  7 19:39 rdkit-doc_202209.3-2_all.deb
    -rw-rw-r--  1 runner runner  99K Mar  7 19:33 rdkit_202209.3-2.debian.tar.xz
    -rw-rw-r--  1 runner runner 2.0K Mar  7 19:33 rdkit_202209.3-2.dsc
    -rw-rw-r--  1 runner runner 2.6M Mar  7 19:40 rdkit_202209.3-2_amd64.build
    -rw-r--r--  1 runner runner  19K Mar  7 19:40 rdkit_202209.3-2_amd64.buildinfo
    -rw-r--r--  1 runner runner 4.1K Mar  7 19:40 rdkit_202209.3-2_amd64.changes
    -rw-r--r--  1 runner runner 1.2K Mar  7 19:40 rdkit_202209.3-2_source.changes
    -rw-rw-r--  1 runner runner  76M Mar  7 19:32 rdkit_202209.3.orig.tar.gz

## Publishing Debian Packages

A `.deb` file on its own is not the most user-friendly. It can be downloaded to
a machine and manually installed with
`dpkg -i librdkit-dev_202209.3-2_amd64.deb` but dpkg can't perform any
dependency resolution. You almost always want to use `apt`, and if you get an
`apt` compatible archive (the convention is managed by `apt-transport-http`)
installation becomes as easy as `apt-get install $pkg`

So what's in an APT HTTP archive? It's a convention of what files exist at what
paths in an HTTP server:

- You don't need any kind of directory listing, just the ability to `HTTP GET`
  files
- A root URI for the archive, for example: `https://ports.ubuntu.com/`
- A distribution/codename which further builds `$ROOT_URI/dists/$CODENAME` like
  `https://ports.ubuntu.com/dists/jammy`
- A repository like `main` which further builds
  `$ROOT_URI/dists/$CODENAME/$REPOSITORY` like:
  `https://ports.ubuntu.com/dists/jammy/main`
- The apt tooling will generate a URI for downloading the full inventory of
  available packages, using the host architecture:
  `$ROOT_URI/dists/$CODENAME/$REPOSITORY/$arch-binary/Packages.gz`
  - This compressed file has the packages, their dependencies names and
    constraints, and relative path information for downloading the deb file. For
    example
    `pool/universe/p/postgresql-common/postgresql-server-dev-all_238_arm64.deb`

We do not want to maintain all these details of this directory layout and
metadata. And each time we add a new package we have to upsert the details in to
`Packages.gz`.

Enter `deb-s3` which handles the repository layout, uploading files, all in to a
commodity-price S3 bucket.
[Read more on their github page](https://github.com/deb-s3/deb-s3). I won't go
over how to authenticate with AWS for S3, or how to create a public-read S3
bucket. But assuming you have credentials setup, this GitHub Action run will
install deb-s3 and upload the `.deb` files to the correct spot:

    # Upload a file to AWS s3
    - name:  Copy index.html to s3
      run: |
        sudo apt-get install ruby awscli
        sudo gem install deb-s3
        aws sts get-caller-identity
        aws s3 ls s3://rdkit-rs-debian/
        deb-s3 upload --bucket=rdkit-rs-debian --s3-region=eu-central-1 --arch=$ARCH --codename=$DIST ../*.deb

After a successful deb build I can use the awscli to list our the S3 contents
(remember our bucket doesn't have directory listing):

    % aws s3 ls --recursive s3://rdkit-rs-debian/
    2023-03-06 17:30:54       2359 dists/jammy/Release
    2023-03-06 17:07:21       9004 dists/jammy/main/binary-amd64/Packages
    2023-03-06 17:30:52       2043 dists/jammy/main/binary-amd64/Packages.gz
    2023-03-06 17:30:53       9004 dists/jammy/main/binary-arm64/Packages
    2023-03-06 17:30:53       2042 dists/jammy/main/binary-arm64/Packages.gz
    2023-03-06 17:07:21          0 dists/jammy/main/binary-armhf/Packages
    2023-03-06 17:30:52         20 dists/jammy/main/binary-armhf/Packages.gz
    2023-03-06 17:07:21          0 dists/jammy/main/binary-i386/Packages
    2023-03-06 17:30:52         20 dists/jammy/main/binary-i386/Packages.gz
    2023-03-06 17:07:19     388980 pool/jammy/l/li/librdkit-dev_202209.3-2_amd64.deb
    2023-03-06 17:30:52     388982 pool/jammy/l/li/librdkit-dev_202209.3-2_arm64.deb
    2023-03-06 17:07:20    4471298 pool/jammy/l/li/librdkit1_202209.3-2_amd64.deb
    2023-03-06 17:30:53    4143838 pool/jammy/l/li/librdkit1_202209.3-2_arm64.deb
    2023-03-06 17:07:20     108884 pool/jammy/p/po/postgresql-15-rdkit_202209.3-2_amd64.deb
    2023-03-06 17:30:53     108884 pool/jammy/p/po/postgresql-15-rdkit_202209.3-2_arm64.deb
    2023-03-06 17:07:20    4369244 pool/jammy/p/py/python3-rdkit_202209.3-2_amd64.deb
    2023-03-06 17:30:53    4350628 pool/jammy/p/py/python3-rdkit_202209.3-2_arm64.deb
    2023-03-06 17:07:20   13159982 pool/jammy/r/rd/rdkit-data_202209.3-2_all.deb
    2023-03-06 17:07:21    7729828 pool/jammy/r/rd/rdkit-doc_202209.3-2_all.deb

Now you can configure a Jammy installation to reference this repository by
configure `apt` with this config file:

    # echo 'deb [trusted=yes] https://rdkit-rs-debian.s3.amazonaws.com jammy main' > /etc/apt/sources.list.d/rdkit-rs.list
    # apt-get update
    Hit:1 http://ports.ubuntu.com/ubuntu-ports jammy InRelease
    Get:2 http://ports.ubuntu.com/ubuntu-ports jammy-updates InRelease [119 kB]
    Hit:3 http://ports.ubuntu.com/ubuntu-ports jammy-backports InRelease
    Get:4 http://ports.ubuntu.com/ubuntu-ports jammy-security InRelease [110 kB]
    Ign:5 https://rdkit-rs-debian.s3.amazonaws.com jammy InRelease
    Get:6 http://ports.ubuntu.com/ubuntu-ports jammy-updates arm64 Contents (deb) [54.0 MB]
    Get:7 https://rdkit-rs-debian.s3.amazonaws.com jammy Release [2359 B]
    Ign:8 https://rdkit-rs-debian.s3.amazonaws.com jammy Release.gpg
    Get:9 https://rdkit-rs-debian.s3.amazonaws.com jammy/main arm64 Packages [2042 B]
    Get:10 http://ports.ubuntu.com/ubuntu-ports jammy-security arm64 Contents (deb) [47.6 MB]
    Fetched 102 MB in 18s (5677 kB/s)
    Reading package lists... Done

Here you can see apt is able to pull out the metadata from our custom archive,
we can ask apt questions about rdkit packages and get references to both the
official Ubuntu releases and our custom release:

    # apt-cache policy librdkit1
    librdkit1:
      Installed: (none)
      Candidate: 202209.3-2
      Version table:
         202209.3-2 500
            500 https://rdkit-rs-debian.s3.amazonaws.com jammy/main arm64 Packages
         202109.2-1build1 500
            500 http://ports.ubuntu.com/ubuntu-ports jammy/universe arm64 Packages

And luckily our rdkit package has the highest priority, so we can install it
without further configuration:

    # apt-get install -y librdkit1
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    The following additional packages will be installed:
      libboost-iostreams1.74.0 libboost-python1.74.0 libboost-serialization1.74.0 libcairo2 libcoordgen3 libmaeparser1 libpixman-1-0 libxcb-render0 libxcb-shm0 libxrender1
    The following NEW packages will be installed:
      libboost-iostreams1.74.0 libboost-python1.74.0 libboost-serialization1.74.0 libcairo2 libcoordgen3 libmaeparser1 libpixman-1-0 librdkit1 libxcb-render0 libxcb-shm0 libxrender1
    0 upgraded, 11 newly installed, 0 to remove and 2 not upgraded.
    Need to get 6102 kB of archives.
    After this operation, 26.7 MB of additional disk space will be used.
    Get:1 http://ports.ubuntu.com/ubuntu-ports jammy/main arm64 libboost-iostreams1.74.0 arm64 1.74.0-14ubuntu3 [243 kB]
    Get:2 https://rdkit-rs-debian.s3.amazonaws.com jammy/main arm64 librdkit1 arm64 202209.3-2 [4144 kB]
    Get:3 http://ports.ubuntu.com/ubuntu-ports jammy/main arm64 libboost-python1.74.0 arm64 1.74.0-14ubuntu3 [293 kB]
    Get:4 http://ports.ubuntu.com/ubuntu-ports jammy/main arm64 libboost-serialization1.74.0 arm64 1.74.0-14ubuntu3 [319 kB]
    Get:5 http://ports.ubuntu.com/ubuntu-ports jammy-updates/main arm64 libpixman-1-0 arm64 0.40.0-1ubuntu0.22.04.1 [160 kB]
    Get:6 http://ports.ubuntu.com/ubuntu-ports jammy/main arm64 libxcb-render0 arm64 1.14-3ubuntu3 [16.2 kB]
    Get:7 http://ports.ubuntu.com/ubuntu-ports jammy/main arm64 libxcb-shm0 arm64 1.14-3ubuntu3 [5848 B]
    Get:8 http://ports.ubuntu.com/ubuntu-ports jammy/main arm64 libxrender1 arm64 1:0.9.10-1build4 [18.8 kB]
    Get:9 http://ports.ubuntu.com/ubuntu-ports jammy/main arm64 libcairo2 arm64 1.16.0-5ubuntu2 [613 kB]
    Get:10 http://ports.ubuntu.com/ubuntu-ports jammy/universe arm64 libcoordgen3 arm64 3.0.0-2 [209 kB]
    Get:11 http://ports.ubuntu.com/ubuntu-ports jammy/universe arm64 libmaeparser1 arm64 1.2.4-1build1 [80.2 kB]
    Fetched 6102 kB in 4s (1725 kB/s)
    [[[ SNIP ]]]

You can see the apt dependency resolver is mixing S3 debian files with official
files. Meaning we are spliced in to the dependency tree just right!

## Multi Architecture Build

The rdkit debian packages are built for both amd64 and arm64 -- we anticipate
deploying to a heterogeneous cloud and we anticipate developers coming from both
Intel laptops and M1 laptops. Docker does include a handy runtime translation
engine, powered by QEMU, but we'd rather avoid expensive and unstable runtime
translation.

We build rdkit twice, in two native environments provided by the paid
[buildjet](https://buildjet.com) service. It is extra overhead to manage the
service and complicate our environment with a paid service but we could not get
around GitHub Actions limitations: no native ARM64 environment so we needed to
use QEMU build environments and a 60 minute build limit which was easily
exceeded.

Even though it wasn't our final configuration, we did learn quite a bit about
emulated cross builds from `pbuilder` and friends. Our notes follow.

### Cross Compiling

True cross compilation would have been preferable, running an amd64-native
binary of GCC to emit code for arm64 -- fast enough to stay inside the free
profile of GitHub Actions. Unfortunately true cross compilation is more than
just installing the GCC binary for cross compilation. It's a full system
configuration nightmare when it comes to libraries. Debian's `apt` installer
doesn't have an easy way of separating platforms -- you can install foreign
architecture libraries in your system with `dpkg --add-architecture arm64` but
you will be mixing and matching amd64 and arm64, keeping architectures straight
is tough.

### Compiling Through QEMU

QEMU is a processor and hardware emulator which lets you run, for example, arm64
on amd64. The runtime instruction translation is transparent but has noticeable
overhead. QEMU forms a great base for building arm64 on amd64 through fancy
sandboxes.

There's a relatively well-documented but still complex constellation of tools
which come together for reproducible build environments inside a Debian host.
Among the many tools, the popular ones: `qemu-user-static`, `chroot`, `binfmt`
(available since Linux 2.1.43), `debootstrap`, and a variety of glue scripts
like `pbuilder` and `cowbuilder`. We can dig in to how to use those tools.

### `binfmt_misc`

Up until this project I assumed QEMU was a full-blown VM with emulation of
system hardware -- but it turns out the Linux kernel allows for an
emulator-per-process architecture. The `binfmt` feature was new to me but the
documentation helps make it clear this is old news:

    Versions 2.1.43 and later of the Linux kernel have contained the binfmt_misc module. This enables a system administrator to register interpreters for various binary formats based on a magic number or their file extension, and cause the appropriate interpreter to be invoked whenever a matching file is executed. Think of it as a more flexible version of the #! executable interpreter mechanism, or as something which can behave a little like "associations" in certain other operating systems (though in GNU/Linux the tendency is to keep this sort of thing somewhere else, like your file manager). update-binfmts manages a persistent database of these interpreters.

You may want to confirm your binfmts is configured correctly in your build
system. You can list out known executable formats and their registered
interpreter:

    update-binfmts --display

And look for aarch64 -- we found this format was not configured in our Ubuntu
docker images. Easy to fix:

    update-binfmts --enable qemu-aarch64

### `debootstrap`

Now any aarch64 executable will automatically get instruction translation and
"just work". The next question is how to set up a pure aarch64 filesystem in
which to run the build. Enter `debootstrap`, a tool for unpacking
Debian-flavored operating systems in to a file system. Think of it like the
usual Debian installer, but it unzips all the `.deb` files in to a directory
tree. Remember in these QEMU examples we're running from an amd64 host
environment. In general you won't need to, but you can run it manually:

    # uname -a
    Linux ruby-ubuntu-build 5.4.149-73.259.amzn2.x86_64 #1 SMP Mon Sep 27 12:48:12 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
    # debootstrap --arch=arm64 --foreign --variant=buildd --verbose --include=fakeroot,build-essential --components=main --resolve-deps --no-merged-usr jammy /tmp/jammy-arm http://ports.ubuntu.com/ubuntu-ports
    I: Retrieving InRelease
    I: Checking Release signature
    I: Valid Release signature (key id F6ECB3762474EDA9D21B7022871920D1991BC93C)
    I: Retrieving Packages
    I: Validating Packages
    I: Resolving dependencies of required packages...
    I: Resolving dependencies of base packages...
    I: Checking component main on http://ports.ubuntu.com/ubuntu-ports...
    I: Retrieving adduser 3.118ubuntu5
    I: Validating adduser 3.118ubuntu5
    I: Retrieving apt 2.4.5
    I: Validating apt 2.4.5
    [[[ SNIP ]]]

Note this `debootstrap` invocation specifies the mirror to use, by default the
bootstrapped OS inherits from the host's archive configuration -- in our case an
amd64 host configured for `archives.ubuntu.com`. Ubuntu does not mix amd64 and
arm64 packages in the same archive, all arm64 packages are relegated to
`ports.ubuntu.com`.

Understanding `debootstrap` may help when debugging high level wrapper scripts
like `pbuilder`.

### `chroot`

From the above example, our `/tmp/jammy-arm` directory is now ready for use. We
can confirm we're in an aarch64 context inside x86 (a unique `uname` signature
for me, up until this point) by _changing root_ with `chroot`. Changing root is
done through a system call, instructing the Linux kernel to change the current
process's root. All file reads after `chroot` are relative to the new root. In
the following example we move our current bash program inside the root and all
`$PATH` work is done from the new root, letting us call the host `uname` and the
guest `uname` in quick succession.

    # uname -a
    Linux ruby-ubuntu-build 5.4.149-73.259.amzn2.x86_64 #1 SMP Mon Sep 27 12:48:12 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
    # chroot /tmp/jammy-arm/
    # uname -a
    Linux ruby-ubuntu-build 5.4.149-73.259.amzn2.x86_64 #1 SMP Mon Sep 27 12:48:12 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux

### Streamlining workflow with `pbuilder`

Pretty cool! But there's no way we want to do the legwork of copying files in to
the chroot environment and running all the `debuild` commands to build a binary
deb package.

The `pbuilder` family of tools have us covered for this
disposable-foreign-filesystem-and-transparent-process-emulation-for-dpkg-buildpackage
workflow. Making it a few high level commands, mostly wrappers for what we
already covered.

### Enabling QEMU's aarch64 in GitHub Actions

You should just use
[docker's qemu enabling action](https://github.com/docker/setup-qemu-action):

    name: ci
    on:
      push:
    jobs:
      qemu:
        runs-on: ubuntu-latest
        steps:
          - name: Set up QEMU
            uses: docker/setup-qemu-action@v2

You can't use the `update-binfmt` in the GitHub Actions workflow sandbox because
the proc filesystem is not exposed. Not sure what exactly the docker action
does, but it works around the issue and things "just work".

### Wrapping `debootstrap` and `cowbuilder` through `pbuilder`

You may have heard of Docker's union filesystem, which provides the ability to
copy-on-write (COW) a docker image's filesystem. The container's filesystem is
built from image layers and the container is allowed to read/write to any file
but it's done in a lazy manner which encourages reuse between containers. COW is
a great technique for quickly, cheaply instantiating sandboxes and I find it
essential when iterating on debian package configurations.

`cowbuilder` is the popular choice for managing a native COW directory tree. But
you won't invoke the tool directly, it's easier to use the wrapper script
`git-pbuilder` which will establish a convention to bridge `cowbuilder`,
`debootstrap`, `pbuilder`, `git` and `dpkg-buildpackage`. That's a lot of layers
in one spot but we can walk through it.

The first step is to create the filesystem, using `debootstrap` and
`cowbuilder`:

    # DIST=jammy ARCH=arm64 git-pbuilder create --mirror=http://ports.ubuntu.com/ubuntu-ports
    I: Invoking pbuilder
    I: forking: pbuilder create --buildplace /var/cache/pbuilder/base-jammy-arm64.cow --mirror http://archive.ubuntu.com/ubuntu/ --architecture arm64 --distribution jammy --no-targz --extrapackages cowdancer

You can see it's putting the cowbuilder filesystem in to `/var/cache/pbuilder`
and naming it based on the architecture and distribution. An enterprising
individual could script this up further to build a debian package for multi
distributions and architectures!

You can still `chroot` in to this image to poke around but the better choice is
to use cowbuilder to remember the before state, giving an easy way to keeping
things pristine for reuse.

    # cowbuilder --login --basepath=/var/cache/pbuilder/base-focal-arm64.cow/
    I: Copying COW directory
    I: forking: rm -rf /var/cache/pbuilder/build/cow.21862
    I: forking: cp -al /var/cache/pbuilder/base-focal-arm64.cow /var/cache/pbuilder/build/cow.21862
    [[[ SNIP ]]]
    # uname -a
    Linux ruby-ubuntu-build 5.4.149-73.259.amzn2.x86_64 #1 SMP Mon Sep 27 12:48:12 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux

Let's keep going with this pristine filesystem and try copying our project in to
the pristine context, running some builds.

### Using git to build images

Before we can get to the magic of moving debian builds inside the COW
filesystem, let's talk about how `git` comes in to all this. Rather than
tracking source code `.tar.gz` files independent of debian config files, which
is fine and has worked for decades, we can instead unify all history in to a
single `git` repository. Understanding and following conventions will help a lot
to walk the happy path with the high-level tooling.

The git repo for the Debian package does not necessarily need to be a fork of
the upstream repository, and based on the upstream's tagging policy it may be
advisable to just start a new repository. You don't want to get confused with
the upstream's tags and you may not want to use all the space of commits between
releases.

The git repo should track all the `debian/` files (`debian/control`,
`debian/changelog`, `debian/rules`, etc). And outside the `debian/` folder
should be the package's source code, all-in-one place. Something like:

    % tree debian-project
    debian-project
    ├── Makefile
    ├── README.md
    ├── debian
    │   ├── changelog
    │   ├── control
    │   └── rules
    └── main.c

Mixing the upstream source code and debian files in a new history seems odd but
it works.

`git-pbuilder` has helpers for intregrating an upstream release's tar/zip file
in to your git repo.
[Take a look at the pbuilder docs for import-orig](http://honk.sigxcpu.org/projects/git-buildpackage/manual-html/gbp.import.fromscratch.html).

That said, let's assume you're fork a repo that's fully usable, you just need
the incantations. Consider this rdkit debian changelog:

    rdkit-debian % head -n 5 debian/changelog
    rdkit (202209.3-2) UNRELEASED; urgency=medium

    * New upstream point release

    -- Debichem Team <debichem-devel@lists.alioth.debian.org>  Sat, 14 Jan 2023 13:33:42 +0100

In parentheses we see `202209.3` and it just so happens there's a corresponding
git tag in the repo:

    % git tag --list | grep 202209.3
    upstream/202209.3-1

This tag differs just barely but that's fine, the stem of `202209.3` (without
the build version `-1` or `-2`) is enough info for `git-pbuilder` to find it and
use it as the source copied in to the cowbuilder filesystem. To be clear: what's
in the changelog when invoking `git-pbuilder` must match tags in the same repo.

Ok with this git convention out of the way we can now start a build inside a
pristine, foreign architecture environment:

### Building the package

Let's start off with the super high-level build command:

    gbp buildpackage --git-ignore-new --git-pbuilder --git-no-pristine-tar --git-arch=arm64 --git-dist=jammy -us -uc

And to break it down:

- `--git-ignore-new` won't care if you have some staged or untracked changes
- `--git-pbuilder` will activate the pbuilder family of tools
- `--git-no-pristine-tar` not sure what this means but when things failed it
  recommended adding the flag
- `--git-arch` set the system architecture, used to find the right `cowbuilder`
  filesystem
- `--git-dist` set the OS distribution, used to find the right `cowbuilder`
  filesystem
- `-us -uc` the usual `dpkg-buildpackage` flags to skip signing the package

Let's kick it all off:

    rdkit# gbp buildpackage --git-ignore-new --git-pbuilder --git-no-pristine-tar --git-arch=arm64 --git-dist=jammy -us -uc
    gbp:info: Building with (cowbuilder) for jammy:arm64
    gbp:info: Creating rdkit_202209.3.orig.tar.gz from 'upstream/202209.3'
    gbp:info: Performing the build
    [[[ SNIP ]]]
    I: Copying COW directory
    I: forking: rm -rf /var/cache/pbuilder/build/cow.12153
    I: forking: cp -al /var/cache/pbuilder/base-jammy-arm64.cow /var/cache/pbuilder/build/cow.12153
    [[[ SNIP ]]]
    I: forking: pbuilder build --debbuildopts  --debbuildopts '  '-us' '-uc'' --buildplace /var/cache/pbuilder/build/cow.12153 --buildresult /home/runner/actions-runner/_work/rdkit-debian --mirror http://ports.ubuntu.com/ubuntu-ports/ --architecture arm64 --distribution devel --no-targz --internal-chrootexec 'chroot /var/cache/pbuilder/build/cow.12153 cow-shell' /home/runner/actions-runner/_work/rdkit-debian/rdkit_202209.3-2.dsc
    [[[ SNIP ]]]
    dpkg-source: info: extracting rdkit in rdkit-202209.3
    Depends: bison, catch2, cmake, debhelper-compat (= 12), dh-python, doxygen, flex, fonts-freefont-ttf, imagemagick [[[ SNIP ]]]
    [[[ SNIP ]]]
    dpkg-deb: building package 'pbuilder-satisfydepends-dummy' in '/tmp/satisfydepends-aptitude/pbuilder-satisfydepends-dummy.deb'.
    [[[[ SNIP ]]]]
    rdkit# ls ..

### Cross build conclusion

There you have it, a nice set of tools and practices for building arm64 inside
of amd64, suitable for a freebie environment like GitHub Actions. With plenty of
context that maybe you can fork a Debian package definition from
[Salsa](https://salsa.debian.org/) and go wild!

## Conclusion

We can now install our custom rdkit in seconds, never having to worry about
build dependencies, those are all relegated to the CI pipeline environment. The
runtime footprint is complete but tight and we can continue to pull in the
wisdom from the DebiChem team while controlling the whole process for our own
needs.

You can see the whole rdkit-debian repository here:
https://github.com/rdkit-rs/rdkit-debian

## Further reading

- http://honk.sigxcpu.org/projects/git-buildpackage/manual-html/gbp.html
- https://en.wikipedia.org/wiki/Chroot
- https://wiki.debian.org/DebianRepository/Format
