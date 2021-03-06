Introduced changes
======================================================================

:Maintainer: Masatake YAMATO <yamato@redhat.com>

----

Many changes have been introduced in universal-ctags. Use git-log to
review changes not enumerated here, especially in language parsers.

Importing most of the changes from exuberant-ctags
---------------------------------------------------------------------
See "exuberant-ctags" in "Tracking other projects" about the status of
importing. Some changes in Fedora and Debian are also imported.

New parsers
---------------------------------------------------------------------
The following parsers have been added:

* ada
* clojure  
* coffee *xcmd*
* css
* d
* ctags option library *optlib*
* falcon
* go
* json
* m4 *optlib*
* mib *optlib*
* rust
* windres
* SystemVerilog

See "Option library" about  *optlib*.
See "External parser command" about *xcmd*.


Heavily improved language parsers
---------------------------------------------------------------------
* php
* verilog


New test facility named *Units*
---------------------------------------------------------------------
The existing test facility *Test* checks for differences between the
output of on older ctags and a newer ctags. If any difference is
found the check fails. *Test* expects the older ctags binary to be
correct.

This expectation is not always met. Consider that a parser for a new
language is added. You may want to add a sample source code for that
language to *Test*. An older ctags version is unable to generate a
tags file for that sample code, but the newer ctags version does. At
this point a difference is found and *Test* reports failure.

On the other hand *Units* compares the expected output that a test
developer prepares with the output of newly built ctags. The expected
output doesn't depend on ctags. For more details see "Using *Units*".

Running tests under valgrind
---------------------------------------------------------------------
With the following command line tests can be run under valgrind memory
checker.

::

	$ make units VG=1 SHELL=/bin/bash

NOTE: ``/bin/bash`` is needed to report the result.


Semi-fuzz testing
---------------------------------------------------------------------
Unexpected input can lead ctags to enter an infinite loop. The fuzz
target tries to identify these conditions by passing
semi-random (semi-broken) input to ctags.

::

	$ make fuzz LANGUAGES=LANG1[,LANG2,...]

With this command line, ctags is run for random variations of all test
inputs under *Units/\*/input.\** of languages defined by ``LANGUAGES``
macro variable. In this target, the output of ctags is ignored and
only the exit status is analyzed. The ctags binary is also run under
timeout command, such that if an infinite loop is found it will exit
with a non-zero status. The timeout will be reported as following::

	[timeout C]                Units/test.vhd.t/input.vhd

This means that if C parser doesn't stop within N seconds when
*Units/test.vhd.t/input.vhd* is given as an input, timeout will
interrup ctags. The default duration can be changed using
``TIMEOUT=N`` argument in *make* command. If there is no timeout but
the exit status is non-zero, the target reports it as following::

	[unexpected-status(N) C]                Units/test.vhd.t/input.vhd

The list of parsers which can be used as a value for ``LANGUAGES`` can
be obtained with following command line

::

	$ ./ctags --list-languages

Besides ``LANGUAGES`` and ``TIMEOUT``, fuzz target also takes the
following parameters:

	``VG=1``

		Run ctags under valgrind. If valgrind finds a memory
		error it is reported as::

			[valgrind-error Verilog]                Units/array_spec.f90.t/input.f90

		The valgrind report is recorded at
		``Units/\*/VALGRIND-${language}.tmp``.

	``SHRINK=1``

		If exit status is non-zero, run ``units shrink`` to
		minimize the bad input. Using bisection algorithm,
		*misc/units shrink* tries to make the shortest input
		which makes ctags exit with non-zero status.
		The result is reported to ``Units/\*/SHRINK-${language}.tmp``.
		This is very useful for debugging.

Automatic parser selection based on corpora
---------------------------------------------------------------------
Ctags has two built-in parsers for suffix *.m*: ``ObjectiveC`` and
``Matlab``. The decision on which parser ctags should use is called
"parser conflict".

Like in ``--language-force`` option, ctags provides some ways to
choose a parser manually. However, it would be nice if ctags could
choose a proper parser without manual instruction.

With ``--<LANG>-corpus=spec:corpusFile`` option you can prepare corpus a
file to make ctags learn lexical tendency of a language. Ctags
learns it as typical input of ``LANG``. Based on this learning ctags
tries to solve the parser conflict. See *Data/optlib/mib.ctags*
and *Data/corpora/RFC1213-MIB.txt* as an example of the usage of
``--<LANG>-corpus``.

For ``ObjectiveC`` and ``Matlab`` parsers, corpus files are embedded
within the parser implementations. See *objc.c* and *matlab.c*.

.. TODO More documentation is needed.


Modeline based parser selection
---------------------------------------------------------------------
exuberant-ctags has the ability to choose a proper parser based on shebang
lines (e.g. *#!/bin/sh*).

Editors like vim and emacs recognize special patterns in files called
modelines. The line is inserted by a user of the text editor and can
be used to set the file type (Vim) or mode (emacs).

universal-ctags also recognizes these modeline and selects a language parser
based on it if ``--guess-language-eagerly`` (or ``-G``) option is given.


ctags recognizes the following patterns used in emacs:

  * at the head of input file or at the line next of shebang line::

      -*- mode: MODE; -*-

    or ::

      -*- MODE -*-

  * at the end of input file::

      Local Variables:
      ...
      mode: MODE
      ...
      End:


ctags recognizes the following patterns used in vim:

  * at the end of input file::

      vim:set filetype=SYNTAX

    or ::

      ex:se ft=SYNTAX


ctags recognizes the following patterns and considers the
input as a zsh script:

  * at the head of input file::

      #autoload

    or ::

      #compdef ....


NOTE: This feature takes a performance hit: it opens the input file
once to detect the file type and a second time to process the file
with the detected parser. For this reason, this feature is enabled
only if the ``--guess-language-eagerly`` option is used. This option
can be placed in the .ctags file to have this feature always enabled.

Better parser selection for template files
---------------------------------------------------------------------
Consider an input file name *foo.c.in*.  Suffix *.in* is popular as a
name for template files.  Well-known one is *config.h.in* used in GNU
autotools.

ctags used suffix here *\*.in* for choosing a parser. *.in* shows
nothing about the language used in the input file. When fishman-ctags
finds *.in* as suffix, fishman-ctags checks the next suffix, here *.c*.

Dry running
---------------------------------------------------------------------
With ``--print-language`` option, you can test the parser selector of
ctags. e.g.::

	$ ./ctags --print-language main.c
	main.c: C

If no parser is selected, ``NONE`` is printed as parser name.


Option library
---------------------------------------------------------------------

exuberant-ctags provides the way to customize ctags with options like
``--langdef=<LANG>`` and ``--regex-<LANG>``. An option file where options are
written can be loaded with ``--options=OPTION_FILE``.

This feature was extended such that ctags treats option files
as libraries. Developers of universal-ctags can maintain option files
as part of universal-ctags, making part of its release. With ``make
install`` they are also installed along with ctags command.

universal-ctags prepares directories where the option files are installed.

Consider a GNU/Linux distribution.
The following directories are searched when loading an option file:

#. *~/.ctags.d/optlib*
#. */etc/ctags/optlib*
#. */usr/share/ctags/optlib*

The name of an option file must have .conf or .ctags as suffix.

If ctags is invoked with following command line::

	$ ctags --options=mib ...

Following files are searched with following order for finding ``mib``:

#.  *~/.ctags.d/optlib/mib.conf*
#.  *~/.ctags.d/optlib/mib.ctags*
#.  */etc/ctags/optlib/mib.conf*
#.  */etc/ctags/optlib/mib.ctags*
#.  */usr/share/ctags/optlib/mib.conf*
#.  */usr/share/ctags/optlib/mib.ctags*

These are called built-in search paths.

If these search paths are not desired, the full path of the option
file can be directly specified with ``--options``. The parameter must
start with */* (absolute path) or *./* (relative path) like::

	$ ctags --option=/home/user/test/mib.cf
	$ ctags --option=./test/mib.cf

Here the suffix restriction doesn't exist.

On GNU/Linux more directories can be added with the environment variable
``CTAGS_DATA_PATH``.

::

	$ CTAGS_DATA_PATH=A:B ctags --options=mib ...

The files are searched with the order described below for finding *mib*:

#. *A/optlib/mib.conf*
#. *A/optlib/mib.ctags*
#. *B/optlib/mib.conf*
#. *B/optlib/mib.ctags*
#. *~/.ctags.d/optlib/mib.conf*
#.  ...

Further more ``--data-path=[+]PATH`` can be used for adding more
directories with environment variable::

	$ CTAGS_DATA_PATH=A:B ctags --data-path=+C --options=mib ...

In this case files are searched with the following order to find
*mib*:

#. *C/optlib/mib.conf*
#. *C/optlib/mib.ctags*
#. *A/optlib/mib.conf*
#. *A/optlib/mib.ctags*
#. *B/optlib/mib.conf*
#. *B/optlib/mib.ctags*
#. *~/.ctags.d/optlib/mib.conf*
#. ...

If *+* is omitted, the directory is set instead of added::

	$ CTAGS_DATA_PATH=A:B ctags --data-path=C --options=mib ...

In this case files are searched with the following order to find
*mib*:

#. *C/config/mib.conf*
#. *C/config/mib.ctags*

The directory list can be emptied using the reserved file name ``NONE``::

	$ CTAGS_DATA_PATH=A:B ctags --data-path=NONE --options=mib ...

In this case ctags only tries to load *./mib*.

See also "Loading option recursively".

How a directory is set/added to the search path can be reviewed using
``--verbose`` option. This is useful for debugging this feature.

Pull requests with updated or new option files are welcome by ctags
developers.

NOTE: Although ``--data-path`` has highest priority, ``--data-path`` doesn't
affect a stage of automatic option file loading. Following files are
automatically loaded when ctags starts:

#. */ctags.cnf* (on MSDOS, MSWindows only)
#. */etc/ctags.conf*
#. */usr/local/etc/ctags.conf*
#. *$HOME/.ctags*
#. *$HOME /ctags.cnf* (on MSDOS, MSWindows only)
#. *.ctags*
#. *ctags.cnf* (on MSDOS, MSWindows only)

NOTE: This feature is still experimental. The name of directories,
suffix rules and other conventions may change.

.. TODO
..
.. * Write about MSWindows more(*.cnf*).
.. * ``accept_only_dot_ctags()`` doesn't  check *.cnf*.

See "Contributing an optlib" if you have a good optlib.

Loading option recursively
---------------------------------------------------------------------

The option file loading rules explained in "Option library" is more
complex. If a directory is specified as parameter for ``--option`` instead
of a file, universal-ctags loads option files under the directory
recursively.

Consider the following command line on a GNU/Linux distribution::

	$ ctags --options=bundle ...

The following directories are searched first:

#. *~/.ctags.d/optlib/bundle.d*
#. */etc/ctags/optlib/bundle.d*
#. */usr/share/ctags/optlib/bundle.d*

If *bundle.d* is found and is a directory, files (*\*.ctags*
and *\*.conf*), directories (\*.d) are loaded recursively.

.. TODO

NOTE: If *bundle.d* is not found above list, file
*bundle.ctags* or *bundle.conf* is searched. This rule is a bit
ugly. Following search rules look better.

#. *~/.ctags.d/optlib/bundle.d*
#. *~/.ctags.d/optlib/bundle.ctags*
#. *~/.ctags.d/optlib/bundle.conf*
#. */etc/ctags/optlib/bundle.d*
#. */etc/ctags/optlib/bundle.ctags*
#. */etc/ctags/optlib/bundle.conf*
#. */usr/share/ctags/optlib/bundle.d*
#. */usr/share/ctags/optlib/bundle.ctags*
#. */usr/share/ctags/optlib/bundle.conf*

NOTE: This feature requires ``scandir`` library function. This feature may
be disabled on which platform scandir is not available. Check ``option-directory``
in the supported features::

	$ ./ctags --list-features
	wildcards
	regex
	option-directory


Directories for preloading
---------------------------------------------------------------------

As written in "Option library", option libraries can be loaded with
``--options`` option. However, loading them without explicitly
specifying it may be desired.

Following files can be used for this purpose.

* ~/.ctags
* /ctags.cnf (on MSDOS, MSWindows only)
* /etc/ctags.conf
* /usr/local/etc/ctags.conf

This preloading feature comes from universal-ctags. However, two
weaknesses exist in this implementation.

* The file must be edited when an option library is to be loaded.

  If one wants to add or remove an ``--options=`` in a *ctags.conf*,
  currently one may have to use sed or something tool for adding or
  removing the line for the entry in */usr/local/etc/ctags.conf* (or
  */etc/ctags.conf*).

  There is a discussion about a similar issue in
  *http://marc.info/?t=129794755000003&r=1&w=2* about */etc/exports*
  of NFS.

* The configuration defined by the system administrator cannot be
  overriden.

  A user must accept all configuration including ``--options=``
  in */etc/ctags.conf* and */usr/local/etc/ctags.conf*.

The following directories were introduced for preloading purpose.

#. *~/.ctags.d/preload*
#. */etc/ctags/preload*
#. */usr/share/ctags/preload*

All files and directories under the directories are loaded recursively,
with two restrictions:

* file/directory name

  The same suffix rules written in "Option library" and
  "Loading option recursively" are applied in preloading, too.

* overriding

  The traversing and loading are done in the order listed above.
  Once a file is loaded, another file with the same name is not loaded.
  Once a directory is traversed, another directory with the same name is
  not traversed.

  universal-ctags prepares */usr/share/ctags/preload/default.ctags*.
  If you want ctags not to load it, make an empty file at
  *~/.ctags/default.ctags*. To customize
  */usr/share/ctags/preload/default.ctags*, copy the file to
  *~/.ctags.d/default.ctags* and edit it as desired.

  Assume */usr/share/ctags/preload/something.d* exits.
  Some *.ctags* files are in the directory. With making
  an empty directory at *~/.ctags.d/something.d*, you
  can make ctags not to traverse */usr/share/ctags/preload/something.d*.
  As the result *.ctags* files under */usr/share/ctags/preload/something.d*
  are not loaded.

  To customize one of file under
  */usr/share/ctags/preload/something.d*, copy
  */usr/share/ctags/preload/something.d* to *~/.ctags.d/somethind.d* recursively.
  Symbolic links can also be used. After copying or symbolic linking, edit
  one of the copied file.

This feature is heavily inspired by systemd.


Long regex flag
---------------------------------------------------------------------

Regex parser is made more useful by adding more kinds of flags
to ``--regex-<LANG>`` expression. As explained in
*ctags.1* man page, ``b``, ``e`` and ``i`` are defined as flags in
exuberant-ctags.

Even if more flags are added like ``x``, ``y``, ``z``,..., users
may not utilize them well because it is difficult to memorize them. In
addition, if many "option libraries" are contributed, we have to
maintain them.

For both users and developers the variety of short flags are just
nightmares.

So universal-ctags now includes an API for defining long flags, which can be
used as aliases for short flags. The long flags requires more typing
but are more readable.

Here is the mapping between the standard short flag names and long flag names:

=========== ===========
short flag  long flag
=========== ===========
b           basic
e           extend
i           icase
=========== ===========

Long flags can be specified with surrounding ``{`` and ``}``.
So the following ``--regex-<LANG>`` expression ::

   --m4-regex=/^m4_define\(\[([^]$\(]+).+$/\1/d,definition/x

is the same as ::

   --m4-regex=/^m4_define\(\[([^]$\(]+).+$/\1/d,definition/{extend}

The characters ``{`` and ``}`` may not be suitable for command line
use, but long flags are mostly intended for option libraries.

The notion for the long flag is also introduced in ``--langdef`` option.

Exclusive flag in regex
---------------------------------------------------------------------

A line read from input files was matched with **all** regular expressions
defined with ``--regex-<LANG>`` (or ``--<LANG>-regex``). Each regular
expression matched successfully emits a tag.

In some cases another policy, exclusive-matching, is preferable to the
all-matching policy. Exclusive-matching means the rest of regular
expressions are not tried if one of regular expressions is matched
successfully,

For specifying exclusive-matching the flags ``exclusive`` (long) and
``x`` (short) were introduced. It is used in *data/optlib/mib.ctags*::


	--mib-regex=/^([^ \t]+)[ \t]+DEFINITIONS ::= BEGIN/\1/d,definitions/{exclusive}
	--mib-regex=/^([a-zA-Z][^ \t]+)[ \t]+[A-Za-z]/\1/n,name/

Another use case for this flag is for ignoring a line::

	--m4-regex=/#.*(define|undefine|s?include)\>//x
	--m4-regex=/\<dnl.*(define|undefine|s?include)\>//x

Comments are started from ``#`` or ``dnl`` in many use case of m4 language.
With above options ctags can ignore ``define`` in comments.

If an empty name pattern(``//``) is found in ``--regex-<LANG>`` option
ctags warns it as wrong usage of the option. However, the flags
``exclusive`` or ``x`` is specified, the warning is suppressed. This
is imperfect approach for ignoring text insides comments but it may
be better than nothing. Ghost kind is assigned to the empty name
pattern. (See "Ghost kind in regex parser".)

Optional flag in regex
---------------------------------------------------------------------

Kinds defined with ``--regex-<LANG>`` (or ``--<LANG>-regex``) was
enabled by default. A kind disabled by default can be defined
with new flag ``optional`` (long only) like::

	--m4-regex=/[[:space:]]include\(`([^']+)'/\1/I,inclusion/x{optional}

A user can turn on this pattern with::

       --m4-kinds=+I


Ghost kind in regex parser
---------------------------------------------------------------------

If a whitespace is used as a kind letter, it is never printed when
ctags is called with ``--list-kinds`` option.  This kind is
automatically assigned to an empty name pattern.

Normally you don't need to know this.

Passing parameter for long regex flag
---------------------------------------------------------------------

In the implemented API long-flags can take a parameters.
Conceptual example::

	--<LANG>-regex=/regexp1/replacement/kind-spec/{transformer=uppercase}
	--<LANG>-regex=/regexp2/replacement/kind-spec/{transformer=lowercase}
	--<LANG>-regex=/regexp2/replacement/kind-spec/{transformer=capitalize}

This is not yet used in any user visible place.
This is implemented for extending ctags in future.

.. TBW

Wildcard in options
---------------------------------------------------------------------

For the purpose gathering as mach as possible information from source
code "wildcard"(``*``) in option is introduced.

``--fields=*``

	Enables all available fields.

``--<LANG>-kinds=*``

	Enables all available kinds for ``LANG``.

``--*-kinds=SPEC``

	Applies SPEC as kinds to all available language parsers.

``--*-kinds=*``

	Enables all available kinds to all available language parsers.

External parser command
---------------------------------------------------------------------

There are commands generating tags file specialized to a language.
`CoffeeTags <https://github.com/lukaszkorecki/CoffeeTags>`_ is an
example. CoffeeTags deals with scripts of coffee language. It is written in
Ruby. Therefore we cannot merge the parser into ctags
directly (Remember ctags written in C). However, the format of tags
file generated by CoffeeTags conforms to `FORMAT
<http://ctags.sourceforge.net/FORMAT>`. This means we can reuse
the output instead of reusing the parser source code.

With the new ``--<LANG>-xcmd=COMMAND`` option, ctags invokes ``COMMAND``
as an external parser command(xcmd) for input files written in
``LANG``. ctags merges the output of ``COMMAND`` into tags file.

By default the following executables are searched with following order for finding
xcmd ``COMMAND``:

#. *~/.ctags.d/drivers/COMMAND*
#. */usr/libexec/ctags/drivers/COMMAND*

These are called built-in search path.

On GNU/Linux more directories can be added with the environment variable
named ``CTAGS_LIBEXEC_PATH``. As same as ``CTAGS_DATA_PATH``,
directories can be set with ``:`` separators to ``CTAGS_LIBEXEC_PATH``.
When searching ``COMMAND``, ctags visits the directories before visiting
the built-in search path.

More search paths can be added with ``--libexec-dir=DIR`` option. ctags
visits ``DIR/drivers`` before visiting the directories specified with
``CTAGS_LIBEXEC_PATH`` and built-in search path. If ctags cannot find
``COMMAND``, ctags treats ``COMMAND`` as an executable file, and tries
to run it.

If an executable file as ``COMMAND`` needs to be specified explicitly,
use absolute (starting with ``/``) or relative path (starting with
``.``) notations.

Generally, an executable file ``COMMAND`` should not be specified
directly because ctags requires very specific behaviors (protocol).
Generally available tags generator like CoffeeTags don't conform with
the expected protocol. Executables under the built-in search
path are expected to fill the gap between generally available tags
generator and universal-ctags. This is the reason why the name
*drivers* is used as part of built-in search path.

To write a driver for a tags generator, please read
-"xcmd protocol and writing a driver".


Notice message and ``--quiet`` option
---------------------------------------------------------------------
There were 3 classes of message in ctags:

*fatal*

	A ciritical error is occured. ctags aborts the execution.

*warning*

	An error is occured but ctags continues the execution.

*verbose*

	Mainly for debugging prupose.


*notice* is a new class of message. It is less important than warning*
*but more important for users than *verbose*. Generally the user can
*ignore *notice*. With ``--quiet`` option can be used to turn off the
priting the *notice* class messages.

Miscellaneous new options
---------------------------------------------------------------------

``--undef[=yes|no]``
    Allows disabling the generation of macro tags from ``#undef``
    directives.


Override the letter for file kind
---------------------------------------------------------------------
(See also #317.)

Background
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
``F`` is widely used as a kind letter for file kind. The ``F`` was
hardcoded in ctags internal. However, we found some built-in parsers
including Ruby uses ``F`` for their own kind. So if you find a tag
having ``F`` as a kind letter, you cannot say what it is well: a
file name or something peculiar in a language. Long kind description
strings may help you but I am not sure all tools utilizing ``tags``
file refer the long kind description strings.

To fix the issue for letters for the kind
we modified ctags as follows:
we let the built-in parsers use ``!`` as a letter for file kind
instead of ``F``.

This modification breaks the backward-compatibility of meaning of tags
file. Forcing to use ``F`` for file kinds to the parsers was another
choice but it also breaks the backward-compatibility. We assumed the
impact of using ``!`` for the parsers may be weaker than forcing
t to use ``F``.

For xcmd and regex parsers we prepare the way to override the default
file kind letter, ``F``. Though using this in regex parser is not
recommend.  Try not using ``F`` as a kind letter in your regex
parser. In xcmd parser you may have no choice if the back-end tags file
generator uses ``F`` for its own purpose.

Usage
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
For overriding add ``fileKind`` long flag ``--langdef=LANG`` option.
Following is an example to use ``Z`` as a kind letter in a language named
``foo``:;

	$ ctags --langdef=foo'{fileKind=Z}' ...

Single quote is used here to avoid the expansion and evaluate the breaths
by shell.

To know the fileKind of languages, use ``--list-file-kind``::

	$ ctags --list-file-kind 
	Ada F
	Ant F
	Asm F
	...
	Ruby !
	...

Build system add possibility to change program name
---------------------------------------------------------------------

As on some systems (e.g. BSD) there is a 'ctags' program in the base
system it's somewhat inconvenient to have the same name for universal-ctags
During ``configure`` you can now change the output executable name.

Add a prefix 'ex' which will result in 'ctags' transformed into 'exctags'

	$ ./configure --program-prefix=ex

Completely change program name, in this case it's important to remember
there is also 'etags' along 'ctags'

	$ ./configure --program-transform-name='s/ctags/my_ctags/; s/etags/myemacs_tags/'
