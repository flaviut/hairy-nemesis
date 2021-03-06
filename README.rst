What is NRE?
============

A regular expression library for Nim using PCRE to do the hard work. The top
priorities are ergonomics & ease of use.

For documentation on how to write patterns, there exists `the official PCRE
pattern documentation
<https://www.pcre.org/original/doc/html/pcrepattern.html>`_. You can also
search the internet for a wide variety of third-party documentation and
tools.

Notes
-----

Issues with toSeq
~~~~~~~~~~~~~~~~~

If you love ``sequtils.toSeq`` we have bad news for you. This library doesn't
work with it due to documented compiler limitations. As a workaround, use this:

.. code-block:: nim

   import nre except toSeq

Licencing
~~~~~~~~~

PCRE has `some additional terms`_ that you must agree to in order to use
this module.

.. _`some additional terms`: http://pcre.sourceforge.net/license.txt

Empty string splitting
~~~~~~~~~~~~~~~~~~~~~~

This library handles splitting with an empty string, i.e. if the splitting
regex is empty (``""``), the same way as `Perl <https://ideone.com/dDMjmz>`__,
`Javascript <http://jsfiddle.net/xtcbxurg/>`__, and `Java
<https://ideone.com/hYJuJ5>`__.

This means that ``"123".split(re"") == @["1", "2", "3"]``, as opposed to the
Nim stdlib's ``@["123"]``

Types
-----

``type Regex* = ref object``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Represents the pattern that things are matched against, constructed with
``re(string)``. Examples: ``re"foo"``, ``re(r"(*ANYCRLF)(?x)foo #
comment".``

``pattern: string``
    the string that was used to create the pattern. For details on how
    to write a pattern, please see `the official PCRE pattern
    documentation.
    <https://www.pcre.org/original/doc/html/pcrepattern.html>`_

``captureCount: int``
    the number of captures that the pattern has.

``captureNameId: Table[string, int]``
    a table from the capture names to their numeric id.


Options
.......

The following options may appear anywhere in the pattern, and they affect
the rest of it.

-  ``(?i)`` - case insensitive
-  ``(?m)`` - multi-line: ``^`` and ``$`` match the beginning and end of
   lines, not of the subject string
-  ``(?s)`` - ``.`` also matches newline (*dotall*)
-  ``(?U)`` - expressions are not greedy by default. ``?`` can be added
   to a qualifier to make it greedy
-  ``(?x)`` - whitespace and comments (``#``) are ignored (*extended*)
-  ``(?X)`` - character escapes without special meaning (``\w`` vs.
   ``\a``) are errors (*extra*)

One or a combination of these options may appear only at the beginning
of the pattern:

-  ``(*UTF8)`` - treat both the pattern and subject as UTF-8
-  ``(*UCP)`` - Unicode character properties; ``\w`` matches ``я``
-  ``(*U)`` - a combination of the two options above
-  ``(*FIRSTLINE*)`` - fails if there is not a match on the first line
-  ``(*NO_AUTO_CAPTURE)`` - turn off auto-capture for groups;
   ``(?<name>...)`` can be used to capture
-  ``(*CR)`` - newlines are separated by ``\r``
-  ``(*LF)`` - newlines are separated by ``\n`` (UNIX default)
-  ``(*CRLF)`` - newlines are separated by ``\r\n`` (Windows default)
-  ``(*ANYCRLF)`` - newlines are separated by any of the above
-  ``(*ANY)`` - newlines are separated by any of the above and Unicode
   newlines:

    single characters VT (vertical tab, U+000B), FF (form feed, U+000C),
    NEL (next line, U+0085), LS (line separator, U+2028), and PS
    (paragraph separator, U+2029). For the 8-bit library, the last two
    are recognized only in UTF-8 mode.
    —  man pcre

-  ``(*JAVASCRIPT_COMPAT)`` - JavaScript compatibility
-  ``(*NO_STUDY)`` - turn off studying; study is enabled by default

For more details on the leading option groups, see the `Option
Setting <http://man7.org/linux/man-pages/man3/pcresyntax.3.html#OPTION_SETTING>`_
and the `Newline
Convention <http://man7.org/linux/man-pages/man3/pcresyntax.3.html#NEWLINE_CONVENTION>`_
sections of the `PCRE syntax
manual <http://man7.org/linux/man-pages/man3/pcresyntax.3.html>`_.

Some of these options are not part of PCRE and are converted by nre
into PCRE flags. These include ``NEVER_UTF``, ``ANCHORED``,
``DOLLAR_ENDONLY``, ``FIRSTLINE``, ``NO_AUTO_CAPTURE``,
``JAVASCRIPT_COMPAT``, ``U``, ``NO_STUDY``. In other PCRE wrappers, you
will need to pass these as seperate flags to PCRE.

``type RegexMatch* = object``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Usually seen as Option[RegexMatch], it represents the result of an
execution. On failure, it is none, on success, it is some.

``pattern: Regex``
    the pattern that is being matched

``str: string``
    the string that was matched against

``captures[]: string``
    the string value of whatever was captured at that id. If the value
    is invalid, then behavior is undefined. If the id is ``-1``, then
    the whole match is returned. If the given capture was not matched,
    ``nil`` is returned.

    -  ``"abc".match(re"(\w)").get.captures[0] == "a"``
    -  ``"abc".match(re"(?<letter>\w)").get.captures["letter"] == "a"``
    -  ``"abc".match(re"(\w)\w").get.captures[-1] == "ab"``

``captureBounds[]: HSlice[int, int]``
    gets the bounds of the given capture according to the same rules as
    the above. If the capture is not filled, then ``None`` is returned.
    The bounds are both inclusive.

    -  ``"abc".match(re"(\w)").get.captureBounds[0] == 0 .. 0``
    -  ``0 in "abc".match(re"(\w)").get.captureBounds == true``
    -  ``"abc".match(re"").get.captureBounds[-1] == 0 .. -1``
    -  ``"abc".match(re"abc").get.captureBounds[-1] == 0 .. 2``

``match: string``
    the full text of the match.

``matchBounds: HSlice[int, int]``
    the bounds of the match, as in ``captureBounds[]``

``(captureBounds|captures).toTable``
    returns a table with each named capture as a key.

``(captureBounds|captures).toSeq``
    returns all the captures by their number.

``$: string``
    same as ``match``


``type RegexInternalError* = ref object of RegexException``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Internal error in the module, this probably means that there is a bug


``type InvalidUnicodeError* = ref object of RegexException``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Thrown when matching fails due to invalid unicode in strings


``type SyntaxError* = ref object of RegexException``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Thrown when there is a syntax error in the
regular expression string passed in


``type StudyError* = ref object of RegexException``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Thrown when studying the regular expression failes
for whatever reason. The message contains the error
code.


Operations
----------

``proc match*(str: string, pattern: Regex, start = 0, endpos = int.high): Option[RegexMatch]``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Like ```find(...)`` <#proc-find>`__, but anchored to the start of the
string. This means that ``"foo".match(re"f").isSome == true``, but
``"foo".match(re"o").isSome == false``.


``iterator findIter*(str: string, pattern: Regex, start = 0, endpos = int.high): RegexMatch``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Works the same as ```find(...)`` <#proc-find>`__, but finds every
non-overlapping match. ``"2222".findIter(re"22")`` is ``"22", "22"``, not
``"22", "22", "22"``.

Arguments are the same as ```find(...)`` <#proc-find>`__

Variants:

-  ``proc findAll(...)`` returns a ``seq[string]``


``proc find*(str: string, pattern: Regex, start = 0, endpos = int.high): Option[RegexMatch]``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Finds the given pattern in the string between the end and start
positions.

``start``
    The start point at which to start matching. ``|abc`` is ``0``;
    ``a|bc`` is ``1``

``endpos``
    The maximum index for a match; ``int.high`` means the end of the
    string, otherwise it’s an inclusive upper bound.


``proc split*(str: string, pattern: Regex, maxSplit = -1, start = 0): seq[string]``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Splits the string with the given regex. This works according to the
rules that Perl and Javascript use:

-  If the match is zero-width, then the string is still split:
   ``"123".split(r"") == @["1", "2", "3"]``.

-  If the pattern has a capture in it, it is added after the string
   split: ``"12".split(re"(\d)") == @["", "1", "", "2", ""]``.

-  If ``maxsplit != -1``, then the string will only be split
   ``maxsplit - 1`` times. This means that there will be ``maxsplit``
   strings in the output seq.
   ``"1.2.3".split(re"\.", maxsplit = 2) == @["1", "2.3"]``

``start`` behaves the same as in ```find(...)`` <#proc-find>`__.


``proc replace*(str: string, pattern: Regex, subproc: proc (match: RegexMatch): string): string``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Replaces each match of Regex in the string with ``subproc``, which should
never be or return ``nil``.

If ``subproc`` is a ``proc (RegexMatch): string``, then it is executed with
each match and the return value is the replacement value.

If ``subproc`` is a ``proc (string): string``, then it is executed with the
full text of the match and and the return value is the replacement
value.

If ``subproc`` is a string, the syntax is as follows:

-  ``$$`` - literal ``$``
-  ``$123`` - capture number ``123``
-  ``$foo`` - named capture ``foo``
-  ``${foo}`` - same as above
-  ``$1$#`` - first and second captures
-  ``$#`` - first capture
-  ``$0`` - full match

If a given capture is missing, ``IndexError`` thrown for un-named captures
and ``KeyError`` for named captures.

``proc escapeRe*(str: string): string``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Escapes the string so it doesn’t match any special characters.
Incompatible with the Extra flag (``X``).


