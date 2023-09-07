iwdevtools
==========

Overview
--------
* `qa-vdb`_ tries to find missing/incorrect shared libraries in ``RDEPEND``
* `qa-cmp`_ compares old and newly installed packages, e.g. list new files
* `qa-sed`_ warns when ``sed`` did nothing, bit like a failing patch would
* `qa-openrc`_ tries to find a few common mistakes in OpenRC init scripts

+ `repo-cd`_ facilitates navigating Gentoo repos' directories with some perks
+ `workdir-cd`_ facilitates navigating Portage temp' directories with some perks
+ `eoldnew`_ emerges the previous version then newest, useful with `qa-cmp`_
+ `scrub-patch`_ removes dirt from patches and may suggest improvements
+ `find-unresolved`_ helps find missing libraries on a stripped embedded system

- `atomf.bashlib`_ provides various pure-bash accurate atom splitting functions
- ``atomf`` is also provided as a simple command to use the above

Available from ``app-portage/iwdevtools`` in Gentoo, see `installing`_.

Tools
=====

qa-vdb
------
Dependencies: portage (portageq), portage-utils (qfile,qlist)

Tries to find issues based on information provided by VDB (``/var/db/pkg``).
Currently this compares ``RDEPEND`` and ``DT_NEEDED`` (from ``scanelf -n``)
for missing dependencies, binding operators, and unspecified slots, then
suggest changes with a diff style output.

Exclusions can be set using config files or command line, either global
or per packages if something is known to be right or irrelevant.

Example output::

    $ qa-vdb xmms2
    VDB: detected possibly incorrect RDEPEND (media-sound/xmms2-0.8_p20161122-r8)
    dev-db/sqlite <
    dev-libs/glib | dev-libs/glib:2
                  > media-libs/libogg
                  > sys-libs/readline:=

Left is current, and right is the suggested replacement.

Says sqlite seems unused despite being in ``RDEPEND`` (xmms2 did implement its
own database backend), and it's linking with libogg and readline with current
``USE`` without ``RDEPEND``. glib -> glib:2 is to suggest explicit ``SLOT``
use when available (can be disabled with ``--no-slot`` among other options).

Alternate output::

    $ qa-vdb --unified gnome-terminal
    VDB: detected possibly incorrect RDEPEND (x11-terms/gnome-terminal-3.40.3)
    +dev-libs/atk
    -dev-libs/libpcre2
    +x11-libs/libX11
    +x11-libs/pango

Note that "unused" dependencies should be taken with a grain of salt, package
may or may not still need it in some other way than ``DT_NEEDED``. ``dlopen()``,
build time checks/headers, executables, and other potential non-library files.

Run ``qa-vdb --help`` or see **qa-vdb(1)** man page for details and
information about known limitations.

qa-sed
------
Wrapper for sed that will notify if files were unmodified by the expression.
Primarily intended to be integrated with portage than used directly.

Example output from portage::

    * Messages for package app-arch/gzip-1.12:

    * SED: the following did not cause any changes
    *     sed -e "s:${EPREFIX}/usr:${EPREFIX}:" -i "${ED}"/bin/gunzip || die
    * no-op: -e s:/usr::

Run ``qa-sed --help`` or see **qa-sed(1)** man page for details.

qa-cmp
------
Dependencies: pax-utils (scanelf), portage (portageq), portage-utils
(qlist), libabigail (abidiff - optional)

Compares an image (i.e. ``/var/tmp/portage/<category>/<package>/image``) with
either another image or installed files, then consolidates differences.
Will display added and removed files, ``DT_SONAME`` changes, ABI changes on
libraries without a new ``DT_SONAME`` (requires ``abidiff`` and debug symbols
for proper checks), and size difference if above a certain threshold.

For filelist differences, by default package version is stripped from
filenames (shows ``*``) to reduce odds of showing uninteresting changes
that aren't *new* files. SONAME list will still show these either way.

Example output from portage (bashrc) while 0.15.1b-r4 is installed::

    # emerge -1 =libid3tag-0.16.1-r1
    [...]
    * CMP: =media-libs/libid3tag-0.15.1b-r4 with media-libs/libid3tag-0.16.1-r1/image
    *  FILES:+usr/lib64/cmake/id3tag/id3tagConfig.cmake
    *  FILES:+usr/lib64/cmake/id3tag/id3tagConfigVersion.cmake
    *  FILES:+usr/lib64/cmake/id3tag/id3tagTargets-gentoo.cmake
    *  FILES:+usr/lib64/cmake/id3tag/id3tagTargets.cmake
    *  FILES:-usr/lib64/libid3tag.so.0
    *  FILES:-usr/lib64/libid3tag.so.0.3.0
    *  FILES:+usr/lib64/libid3tag.so.*
    * SONAME:-libid3tag.so.0(64)
    * SONAME:+libid3tag.so.0.16.1(64)
    * ------> FILES(+5,-2) SONAME(+1,-1)

It can pick the two latest ``ebuild install`` for a package and ignore
the system's copy with ``-I/--image-only``, so for a direct-use qa-cmp
example that's also using ``abidiff`` for `bug #616054`_::

    # ebuild libcdio-paranoia-0.93_p1-r1.ebuild clean install
    # ebuild libcdio-paranoia-0.94_p1.ebuild clean install
    # qa-cmp -I libcdio-paranoia
    CMP: dev-libs/libcdio-paranoia-0.93_p1-r1/image with dev-libs/libcdio-paranoia-0.94_p1/image
     FILES:-usr/share/doc/libcdio-paranoia-${PV}/README.zst
     FILES:+usr/share/doc/libcdio-paranoia-${PV}/README.md.zst
       ABI: libcdio_cdda.so.2(64) func(+12,-25) vars(+3) [BREAKING]
       ABI: libcdio_paranoia.so.2(64) func(+47,-10) vars(+3,-1) [BREAKING]
    ------> FILES(+1,-1) ABI(+65,-36,>B<)

.. _bug #616054: https://bugs.gentoo.org/616054

Note that ``[BREAKING]`` doesn't necessarily mean there's a problem
(e.g. may have removed private functions that nothing was using), but
all revdeps *built against the old library* should really be tested
after the upgrade.

Since version 0.10.0 it also checks for permission changes, but may be
a bit quirky depending on how the system handles permissions as they
can't be read from VDB. If running into too many false positives, may
want to use ``--ignore-perms``. After USE=-suid on util-linux::

    * CMP: =sys-apps/util-linux-2.37.2-r3 with sys-apps/util-linux-2.37.2-r3/image
    *  FILES:-bin/mount (-rws--x--x root:root)
    *  FILES:+bin/mount (-rwxr-xr-x root:root)
    *  FILES:-bin/umount (-rws--x--x root:root)
    *  FILES:+bin/umount (-rwxr-xr-x root:root)
    * ------> FILES(+2,-2)

Run ``qa-cmp --help`` or see **qa-cmp(1)** man page for details and
information about known limitations.

qa-openrc
---------
Dependencies: portage (portageq), portage-utils (qlist)

Tries to find common mistakes in OpenRC service scripts.

Example output::

    $ qa-openrc =net-print/cups-2.3.3_p2-r3
    OPENRC: unnecessary usage of start_stop_daemon_args found:
    cupsd: -m should be replaced with command_background=yes
    cupsd: --pidfile should be replaced with pidfile="/run/cupsd.pid"

Run ``qa-openrc --help`` or see **qa-openrc(1)** man page for details.

repo-cd
-------
Dependencies: portage-utils (q), libxml2 (xmllint)

Can be used to jump to the repo directory (cd) of the specified atom,
with a few added perks like displaying ``metadata.xml``'s remote-ids.

Here I have my work tree at ``~/gentoo`` that I want to use with ``:default:.``
as fallback (adds all from ``repos.conf``, and current directory if a repo),
using a ``rcd`` alias::

	~$ eval "$(command repo-cd --bash=rcd --path="~/gentoo:default:.")"
	~$ rcd speed-d<tab>
	 > ~/gentoo/games-sports/speed-dreams
	 D Fork of the famous open racing car simulator TORCS (2.2.3)
	 H http://www.speed-dreams.org/ (2.2.3)
	 H https://sourceforge.net/projects/speed-dreams/
	 G https://packages.gentoo.org/packages/games-sports/speed-dreams
	 G https://bugs.gentoo.org/buglist.cgi?quicksearch=games-sports%2Fspeed-dreams
	 M games@gentoo.org
	 + Manifest
	 + files
	 + metadata.xml
	 + speed-dreams-2.2.3.ebuild
	~/gentoo/games-sports/speed-dreams$ _

Has some customization options, like hiding fields and running commands in
the directory (in the above case it ran ``ls`` by default), and these can
be saved in a ``repo-cd.conf`` or like ``--path`` was above::

	~$ rcd zstd --fields=all,-bgo,-pgo,-maint --run="echo hello world"
	 ? 1:~/gentoo/app-arch/zstd (default)
	 ? 2:~/gentoo/dev-python/zstd
	 ? Choice? 2
	 > ~/gentoo/dev-python/zstd
	 D Simple python bindings to Yann Collet ZSTD compression library (1.5.2.5)
	 H https://github.com/sergey-dryabzhinsky/python-zstd/
	 H https://pypi.org/project/zstd/
	 + hello world
	~/gentoo/dev-python/zstd$ _

For a more involved ``--run`` example, in ``~/my-gentoo-notes``, have::

	dxvk:
	- remember that thing next bump you silly goose
	- also this, you always forget to do it
	vkd3d-proton:
	- some other stuff

Then in ``~/my-repo-cd-cmd`` (with ``chmod +x``):

.. code-block:: bash

	#!/usr/bin/env bash
	echo -e "\e[094mhttps://blue-extra-link/?search=${RCD_CATEGORY}%2F${RCD_PN}"

	pkgdev showkw --color=y # from dev-util/pkgdev

	# use sed to show lines after 'package-name:' in red if starts with dash,
	# could also use simple options like [[ -f path/${RCD_PACKAGE} ]] && cat ...
	red=$'\e[091m'
	sed -n "/^${RCD_PN}:/,/^[^-].*:/{s/^-/${red}-/p}" ~/my-gentoo-notes

In ``~/.config/iwdevtools/repo-cd.conf`` (also see ``--dumpconfig`` option)::

	run=~/my-repo-cd-cmd

Results in::

	~$ rcd dxvk --fields=dir,bgo
	 > /var/db/repos/gentoo/app-emulation/dxvk
	 G https://bugs.gentoo.org/buglist.cgi?quicksearch=app-emulation%2Fdxvk
	 + https://blue-extra-link/?search=app-emulation%2Fdxvk
	 +         a a   a     l       p r   s
	 +         l m   r h i o m m   p i s p   e s r
	 +         p d a m p a o 6 i p c s 3 a x a l e
	 +         h 6 r 6 p 6 n 8 p p 6 c 9 r 8 p o p
	 +         a 4 m 4 a 4 g k s c 4 v 0 c 6 i t o
	 + ------------------------------------------------
	 +  1.10.3 * ~ * * * * * * * * * * * * ~ 8 0 gentoo
	 +  9999   o o o o o o o o o o o o o o o 8 0 gentoo
	 + - remember that thing next bump you silly goose
	 + - also this, you always forget to do it
	/var/db/repos/gentoo/app-emulation/dxvk$ _

Run ``repo-cd --help`` or see **repo-cd(1)** man page for details and
information about known limitations.

workdir-cd
-------
Dependencies:  portage (portageq), libxml2 (xmllint)

Can be used to jump to the working directory (cd) of the specified atom.

If ``--path=default`` workdir-cd will try to get the path from ``protageq``.
If nothing is set as PORTAGE_TMPDIR in make.conf, the usual location of
TMPDIR is ``/var/tmp/portage``.

I unpack two packages using the ``ebuild`` command and then use the workcd
alias without an argument to navigate to the latest workdir in PORTAGE_TMPDIR::

  ~/dev/gentoo/app-admin/entr $ eval "$(command workdir-cd --bash=workcd --path="default")"
  ~/dev/gentoo/app-admin/entr $ ebuild entr-5.3-r1.ebuild unpack
  Appending /home/pascal/dev/gentoo to PORTDIR_OVERLAY...
  * entr-5.3.tar.gz BLAKE2B SHA512 size ;-) ...                                                                 [ ok ]
  >>> Unpacking source...
  >>> Unpacking entr-5.3.tar.gz to /var/tmp/portage/app-admin/entr-5.3-r1/work
  >>> Source unpacked in /var/tmp/portage/app-admin/entr-5.3-r1/work
  ~/dev/gentoo/app-admin/entr $ ebuild entr-5.4.ebuild unpack
  Appending /home/pascal/dev/gentoo to PORTDIR_OVERLAY...
  * entr-5.4.tar.gz BLAKE2B SHA512 size ;-) ...                                                                 [ ok ]
  >>> Unpacking source...
  >>> Unpacking entr-5.4.tar.gz to /var/tmp/portage/app-admin/entr-5.4/work
  >>> Source unpacked in /var/tmp/portage/app-admin/entr-5.4/work
  + LICENSE
  + Makefile.bsd
  + Makefile.linux
  + Makefile.macos
  + NEWS
  + README.md
  + configure
  + data.h
  + entr.c
  + entr.1
  + missing
  + system_test.sh
  /var/tmp/portage/app-admin/entr-5.4/work/entr-5.4 $

If it is given a string other than '-' as the argument, workdir-cd will list
the found workdirs matching that argument::

 / $ workcd entr
 ? 1:/var/tmp/portage/app-admin/entr-5.3-r1 (default)
 ? 2:/var/tmp/portage/app-admin/entr-5.4
 ? Choice?

When the alias is ran with '-' as the argument, it will list all available workdirs
in tmpdir and ask for which one to cd into::

 / $ workcd -
 ? 1:/var/tmp/portage/app-admin/entr-5.4 (default)
 ? 2:/var/tmp/portage/app-admin/entr-5.3-r1
 ? 3:/var/tmp/portage/x11-misc/xscreensaver-6.06-r2
 ? 4:/var/tmp/portage/x11-misc/xscreensaver-6.07
 ? 5:/var/tmp/portage/sys-apps/systemd-253.6
 ? Choice?

Has some customization options, like hiding fields and running commands in
the directory (in the above case it ran ``ls`` by default), and these can
be saved in a ``workdir-cd.conf`` or like ``--path`` was above::

 /  $ workcd entr --run="echo hello world"
 ? 1:/var/tmp/portage/app-admin/entr-5.3-r1 (default)
 ? 2:/var/tmp/portage/app-admin/entr-5.4
 ? Choice? 1
 + hello world

Run ``workdir-cd --help`` or see **workdir-cd(1)** man page for details and
information about known limitations.

eoldnew
-------
Dependencies: portage (portageq)

Helper for using ``qa-cmp`` which emerges a package for a given atom but
by first emerging its previous (visible) version if not already installed.

Example usage::

    $ eoldnew iwdevtools --quiet --pretend
    old: app-portage/iwdevtools-0.1.1
    new: app-portage/iwdevtools-0.2.0
    running: emerge =app-portage/iwdevtools-0.1.1 --quiet --pretend
    [ebuild  N    ] app-portage/iwdevtools-0.1.1
    running: emerge iwdevtools --quiet --pretend
    [ebuild  N    ] app-portage/iwdevtools-0.2.0

Run ``eoldnew --help`` or see **eoldnew(1)** man page for details.

scrub-patch
-----------
Perhaps copying the ``sed`` from the `devmanual`_ was too much of a hassle?
Well this is the script for you!

.. _devmanual: https://devmanual.gentoo.org/ebuild-writing/misc-files/patches/index.html

May possibly do a bit more...

Run ``scrub-patch --help`` or see **scrub-patch(1)** man page for details.

find-unresolved
---------------
Dependencies: pax-utils (scanelf)

Scan a ``ROOT`` path's ELF files for missing soname dependencies.
Primarily intended for verification of a stripped embedded system::

    $ find-unresolved netboot-hppa32-20200319T011207Z/
     * Scanning netboot-hppa32-20200319T011207Z for unresolved soname dependencies...
    bin/nano:libtinfow.so.6
    sbin/swapon:libsmartcols.so.1
    sbin/sfdisk:libfdisk.so.1 libsmartcols.so.1 libreadline.so.7
    <snip>
     * Found 6 missing libraries:
       - libfdisk.so.1
       - libtinfow.so.6
    <snip>

Run ``find-unresolved --help`` or see **find-unresolved(1)** man page
for details.

Bashlibs
========

Primarily intended for internal use, but exposing for anyone that may need.
May potentially be subject to breaking changes for the time being.

atomf.bashlib
-------------

Pure bash functions to split portage atoms and version strings. Similar
functionality to **qatom(1)** but is intended to ease usage in bash scripts.

.. code-block:: bash

	#!/usr/bin/env bash
	. "$(pkg-config iwdevtools --variable=atomf)" || exit

	atomf 'ver:%V rev:%R\n' 'cat/pn-1.0-r1' # ver:1.0 rev:1

	atomset 'cat/pn-1.0-r1:slot'
	echo "${CATEGORY},${PN},${PV},${SLOT}" # cat,pn,1.0,slot

	atoma myassocarray '>=cat/pn-1.0-r1:3/stable'
	echo "sub:${myassocarray[subslot]}" # sub:stable

	pversp myarray '1.0b_alpha3_p8-r1'
	echo "${myarray[*]}" # 1 .0 b _alpha 3 _p 8 -r1

Can also use the command line frontend::

	$ atomf 'cat:%c name:%n pvr:%v%r\n' */*/*.ebuild
	cat:acct-group/ name:abrt pvr:-0-r1
	[...]

Run ``atomf --help`` or see **atomf(1)** man page for details.

Installing
==========

On Gentoo, simply ``emerge app-portage/iwdevtools``

Or for a manual install:

- ``mkdir build && cd build``
- ``meson setup --prefix /path/to/prefix``
- ``meson test``
- ``meson install``

To (optionally) integrate with portage, an example bashrc will be installed
at ``<prefix>/share/iwdevtools/bashrc`` which can be either symlinked to or
sourced from ``/etc/portage/bashrc``. See ``--help`` or man pages of commands
for further information and environment options.

Note: normal Gentoo ecosystem is generally expected (e.g. have a typical system
set with GNU coreutils rather than plain POSIX tools). Also, only portage is
supported at the moment as some tools rely on non-standard elements like
VDB/tempdir structure and alternate package managers (e.g. pkgcore) may cause
erroneous reports among other issues.
