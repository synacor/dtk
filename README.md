DTK (<strong>d</strong>ata <strong>t</strong>ool<strong>k</strong>it) is a suite of tools for parsing, analyzing, and graphing logs and other datasets.

In short, DTK converts *data* into *knowledge*.

<sub>*Note:* In the examples in this document, a prefix of ``$`` indicates that the line is a command at a ``bash``-like shell prompt, and a token like ``<count>`` is a placeholder for some actual value named "count".</sub>

# Introduction

Are your logs just a huge pile of data you lug around because "you might need them some day"?  Do you make decisions without the knowledge locked away in those files because it's cumbersome to get from an Apache log to user behavior statistics?  Do you waste hours digging through logs to track down the root cause of an incident?  You're in the right place.  Have a seat.

The modules in DTK follow the [Unix philosophy](http://en.wikipedia.org/wiki/Unix_philosophy) - they do one thing and do it well, they work together, and they operate on text streams.

DTK provides many tools called "modules"; much like [Git](http://git-scm.com/), they are all accessible through the launcher ``dtk`` - ``dtk help``, ``dtk filter``, ``dtk parse``, etc.  Each module focuses on solving one kind of problem while accepting and producing data in a [reusable, common format](http://en.wikipedia.org/wiki/Tab-separated_values).

Because of this, a DTK workflow often involves a [pipe chain](http://en.wikipedia.org/wiki/Pipeline_%28Unix%29) consisting of both DTK modules and other programs.  These can be simple or complex:

```
# flags and other arguments omitted for brevity:

$ zcat | dtk parse | dtk hist1

$ zcat | grep | cut | cut | dtk filter | dtk uc | dtk filter | dtk hist2_bycol

$ cat <(zcat | dtk parse | dtk filter) <(zcat | dtk parse | dtk filter) | dtk delta1
```

Some DTK pipelines work like [map-reduce](http://en.wikipedia.org/wiki/MapReduce) jobs, and, in fact, the DTK suite is also quite effective when run in true [big data](http://en.wikipedia.org/wiki/Big_data) map-reduce environments such as [Hadoop Streaming](http://hadoop.apache.org/docs/stable/streaming.html).

DTK prioritizes usefulness and efficiency over shiny interfaces and PowerPoint presentations.  These aren't your boss' tools (unless he also likes this sort of thing, of course).

# ``dtk``

The main ``dtk`` launcher, which is normally at ``/usr/bin/dtk``, is responsible for finding and invoking the individual modules, which are normally in ``/usr/libexec/dtk-modules``. To invoke a module, use ``dtk <module>``, much as you would with ``git``. If you would like to load modules from a different directory (such as during ``dtk`` development or in a homedir-local install), set the ``DTK_MODPATH`` environment variable to a colon-delimited list of module directories, much like the ``PATH`` environment variable.  An empty string as one of the parts of ``DTK_MODPATH`` will be replaced with the default (``/usr/libexec/dtk-modules``), making it easy to provide overlay directories by using something like ``:~/dtk-modules`` (which would expand to ``/usr/libexec/dtk-modules`` and ``~/dtk-modules``).

To get details on a specific module, use ``dtk help <module>``, which simply invokes ``dtk <module> --help`` for you.

Some modules (like ``hist2`` and ``plot``) work better when more colors are available. For best results, get a terminal with 256-color support, and then set your ``TERM`` environment variable to ``xterm-256color``. If you are within screen, you may have to do this explicitly, like ``cat stuff | TERM=xterm-256color dtk hist2``.

# Modules

Most of the tools produce or operate on newline-terminated records containing tab-delimited fields, often called TSV ("<strong>t</strong>ab-<strong>s</strong>eparated <strong>v</strong>alues"). Unless otherwise specified, tools in DTK take input on STDIN and produce output on STDOUT.

## ``help``

Merely invoking ``dtk help`` produces a list of all available modules.

To get full documentation on a specific module, use ``dtk help <module>`` (like ``dtk help uc``) which just invokes ``dtk <module> --help`` for you.

## ``filter``

The ``filter`` module is similar to ``perl -ne`` - it runs its first argument as Perl code wrapped in some convenience logic.  The given code is run for each record after filling ``@v`` with each field; the final value of ``@v`` after the code is run is used as the output record. Use ``next`` to drop a record or ``last`` to skip all remaining records.

For example, suppose you have a dataset containing pairs of numbers:

```
$ perl -e 'print int(rand()*100)."\t".int(rand()*100)."\n" for 1..100' > data
$ head -n 5 data
50      34
13      15
37      5
62      2
7       88
```

You could add a column which contains the sum of the first two:

```
$ cat data | dtk filter '$v[2] = $v[0] + $v[1]' | head -n 5
50      34      84
13      15      28
37      5       42
62      2       64
7       88      95
```

Or perhaps you only want records where the first field is less than 20 and the second field is more than 80:

```
$ cat data | dtk filter 'next unless $v[0] < 20 && $v[1] > 80'
7       88
10      86
8       82
2       82
12      87
0       91
14      95
```

Because DTK is commonly used to analyze log files, helper functions are also available for fast parsing of data often found in logs:

<dl>
<dt><code>parse_uri</code></dt>
<dd>Takes a URI (or part of one, like <code>/path/to/thing?a=1&b=2</code> or <code>file:///path/to/thing.doc</code>) and returns an empty list on failure or otherwise a hash containing <code>schema</code>, <code>auth</code>, <code>host</code>, <code>port</code>, <code>path</code>, <code>query</code>, and <code>fragment</code>.</dd>

<dt><code>parse_query</code></dt>
<dd>Takes a query string (key/value pairs in the <code>application/x-www-form-urlencoded</code> format) and returns a hash containing the key/value pairs. Keys with no <code>=value</code> part will receive the value <code>1</code>, such that <code>a&b=2</code> will return <code>(a=>1, b=>2)</code>.</dd>

<dt><code>parse_cookies</code></dt>
<dd>Takes a cookie string (key/value pairs like <code>a=1; b=2; c=3</code>) and returns a hash containing the key/value pairs.</dd>

<dt><code>decode_uri</code></dt>
<dd>Takes a URI-encoded string (containing codes like <code>%2d</code> for <code>=</code>) and returns the decoded string.</dd>
</dl>

Here are some example URIs and the result of ``parse_uri``:

```
$ cat uris
http://www.google.com/path/to/page.cgi?a=1&b=2&c#jump
http://www.google.com/
/path/to/thing?a=1&b=2
file:///path/to/thing.doc

$ cat uris | dtk filter 'my %u = parse_uri($v[0]); @v = ($u{host}, $v[0])'
www.google.com  http://www.google.com/path/to/page.cgi?a=1&b=2&c#jump
www.google.com  http://www.google.com/
                /path/to/thing?a=1&b=2
                file:///path/to/thing.doc

$ cat uris | dtk filter 'my %u = parse_uri($v[0]); @v = ($u{query}, $v[0])'
a=1&b=2&c       http://www.google.com/path/to/page.cgi?a=1&b=2&c#jump
                http://www.google.com/
a=1&b=2         /path/to/thing?a=1&b=2
                file:///path/to/thing.doc
```

Here is a filter which shows the query string parameters ``b`` and ``c`` when parameter ``a`` has a truthy value:

```
$ cat uris | dtk filter 'my %u = parse_uri($v[0]); my %q = parse_query($u{query}); next unless $q{a}; @v = ($q{b}, $q{c}, $v[0])'
2       1       http://www.google.com/path/to/page.cgi?a=1&b=2&c#jump
2               /path/to/thing?a=1&b=2
```

## ``parse``

The ``parse`` module parses specific file formats and produces the fields of each record in DTK-friendly tab-delimited output. To get a list of known formats, use ``dtk parse --help`` (or ``dtk help parse``). To see details on a specific format, use ``dtk parse --help <format>`` (or ``dtk help parse <format>``).

Parse formats are executable (``chmod +x``) Perl scripts discovered in the ``parse-formats`` subdirectory of any path given in ``DTK_MODPATH``, which by default causes a search only in ``/usr/libexec/dtk-modules/parse-formats/``.  The following parse formats are packaged with DTK:

<dl>
<dt><code>apache_access</code></dt>
<dd>For parsing the default Apache access logs in the built-in <code>common</code> or <code>combined</code> log formats.</dd>
</dl>

To parse a format, pipe it to ``dtk parse <format>`` to get all fields, or ``dtk parse <format> <field>,<field>,...`` to get specific fields.  For example:

```
$ cat access_log | dtk parse apache_access datetime,bytes
09/May/2012:16:00:00 -0400      -
09/May/2012:16:00:00 -0400      -
09/May/2012:16:00:00 -0400      43
09/May/2012:16:00:00 -0400      -
09/May/2012:15:59:59 -0400      -
09/May/2012:16:00:00 -0400      -
09/May/2012:15:59:59 -0400      931
09/May/2012:16:00:00 -0400      43
09/May/2012:15:59:59 -0400      179090
09/May/2012:16:00:00 -0400      515
```

Some fields have extra post-filters which can be applied by specifying the field as ``<field>:<filter>`` - these are shown in the format details in parentheses after each field. For example, ``apache_access``'s ``datetime`` field can be converted to epoch time, ``bytes`` can be forced to a number (so ``-`` becomes ``0``), and ``useragent`` can be categorized into general classes for easy bucketing:

```
$ cat access_log | dtk parse apache_access datetime:epoch,bytes:numeric,useragent:class
1336593600      0       ie8
1336593600      0       ie8
1336593600      43      ie9
1336593600      0       ie8
1336593599      0       ie9
1336593600      0       ie9
1336593599      931     firefox
1336593600      43      ie9
1336593599      179090  ie7
1336593600      515     ie9
```

### Parse format details

Each parse format is merely an executable (``chmod +x``) Perl script which returns a data structure representing instructions on what fields are available, how to build a regular expression which extracts those fields, and any filters that can be applied to the values in those fields before returning them.  These instructions are represented as an array reference containing either a string (to be skipped as a literal delimiter at that position in each record) or an array reference containing the field's name, regex, and optional filters.  When loaded, ``parse`` will determine from this data the minimum regular expression  required to extract the requested fields.

Suppose your application creates logs like this:

```
[2000-01-01 12:00:42 GMT] INFO - "system" Application starting
[2000-01-01 12:03:04 GMT] Warn - "Bob Johnson" Wrong password
[2000-01-01 12:03:18 GMT] INFO - "Bob Johnson" User has connected
[2000-01-01 12:12:53 GMT] critical - "Bob Johnson" Home directory full
[2000-01-01 13:45:08 GMT] INFO - "system" Home directory cleanup script started
```

This log is messy, weird, and exactly what you might find in the real world.  Let's parse it with DTK by making a new ``weirdlog`` parse format in the ``parse-formats`` directory of one of our module directories and set it to executable (``chmod +x parse-formats/weirdlog``).  Here's our first attempt:

```perl
use warnings;
use strict;

return [
  '[',
  [datetime  => qr/[^\]]+/],
  '] ',
  [level     => qr/\S+/   ],
  ' - "',
  [user      => qr/[^\"]+/],
  '" ',
  [message   => qr/.+/    ],
];
```

This parse format gives us four fields and returns them verbatim from the log.  We can see the fields by asking DTK about them:

```
$ dtk help parse weirdlog
Fields (and filters) for format 'weirdlog' are:
  datetime
  level
  user
  message
```

Then, we can pull the log apart into DTK-friendly tab-separated format:

```
$ cat weirdlog | dtk parse weirdlog
2000-01-01 12:00:42 GMT   INFO      system       Application starting
2000-01-01 12:03:04 GMT   Warn      Bob Johnson  Wrong password
2000-01-01 12:03:18 GMT   INFO      Bob Johnson  User has connected
2000-01-01 12:12:53 GMT   critical  Bob Johnson  Home directory full
2000-01-01 13:45:08 GMT   INFO      system       Home directory cleanup script started
```

Now, let's update the parse format with some filters to help clean the data up a little:

```perl
use warnings;
use strict;

use Date::Parse;  #provides str2time() for human-to-epoch date parsing

return [
  '[',
  [datetime  => qr/[^\]]+/, {epoch=>\&str2time}],
  '] ',
  [level     => qr/\S+/,    {lowercase=>sub{ lc $_[0] }, uppercase=>sub{ uc $_[0] }}],
  ' - "',
  [user      => qr/[^\"]+/],
  '" ',
  [message   => qr/.+/    ],
];
```

We can see our new filters in the field list:

```
dtk help parse weirdlog
Fields (and filters) for format 'weirdlog' are:
  datetime (epoch)
  level (uppercase, lowercase)
  user
  message
```

Let's use our new filters and also rearrange the fields for good measure:

```
$ cat weirdlog | dtk parse weirdlog user,level:lowercase,datetime:epoch
system       info      946728042
Bob Johnson  warn      946728184
Bob Johnson  info      946728198
Bob Johnson  critical  946728773
system       info      946734308
```

Keeping timestamps in epoch time has significant advantages when used with the histogram tools like ``hist1`` and ``hist2``.

## ``plot``

This tool will draw simple 2d scatterplots. Its input should consist of two-column data containing x and y coordinates; optionally, a third column can be provided which indicates the color (1-7) to use for the point. If the viewport bounds are not specified, all data will be read before any output is produced so the viewport can be automatically adjusted to fit the whole dataset. If all viewport bounds are given, the scatterplot will be drawn as data is read, allowing, for example, an animated chart from a ``tail -f`` of an access log. Extents can either be given together as ``view=<xl>..<xh>,<yl>..<yh>`` or individually as ``xl=<position>``, ``yh=<position>``, etc.

Here is a plot of ``sin(x)`` and ``cos(x)``:

```
$ perl -e 'for ($x=-5;$x<5;$x+=.1) {
             print join("\t", $x, sin($x), 1)."\n";
             print join("\t", $x, cos($x), 2)."\n";
           }' \
  | dtk plot
```

![plot example 1](https://raw.github.com/synacorinc/dtk/master/doc-images/example_plot_001.png)

This can be especially useful for things like watching the relationship between two values in the logs in realtime; we use this for things like graphing logs that contain, for example, request time vs bytes returned.  Here's an example that will animate drawing a spiral in your terminal:

```
$ perl -e 'use Time::HiRes qw(sleep);
           $| = 1;
           for ($t=0; $t<10; $t+=.01) {
             print join("\t", $t*cos($t), $t*sin($t)) . "\n";
             sleep .01;
           }' \
  | dtk plot view=-10..10,-10..10 flush_size=1
```

(The ``$| = 1`` line turns on autoflush so the script prints every line individually; ``flush_size=1`` turns off input buffering so each point is rendered, which is only sane for reading slow streams.)

## ``sort``

This module takes rows of tab-delimited data on STDIN and rules about how to index and sort the records as command-line arguments.  Each row is broken into columns in `@v` and converted to sortable keys by each rule.  The rows are sorted by the keys given by the rules and flags, giving earlier rules and flags a higher precedence.

Each rule can be a number (to simply sort by that column, starting from index 0), a macro (of the form ``<column>:<macro>``; see below), or Perl code.  Code rules are executed in the context of a do{} block and should produce a sort key by evaluating it as its last expression.  Specifying only a number is like specifying code which simply returns that field; that is, ``3`` is the same as ``'$v[3]'``.  For example, to sort by the third column (``2``) and then sort rows with matching third-column values by their first column (``0``), pipe TSV data to ``dtk sort 2 0``.  Or, you could sort by the sum of the second (``1``) and third (``2``) column by piping data to ``dtk sort '$v[1]+$v[2]'``.

Flags set modes which apply to subsequent rules until another mode overrides the previous.  They consist of:
- either ``-n`` or ``-s`` to switch to a numeric (``<=>``) or string (``cmp``) comparison
- either ``-a`` or ``-d`` to switch to ascending or descending sort

These flags apply to all subsequent rules until they are overridden by a different flag of the same type.  (The default is to sort numeric/ascending as if ``-an`` had been specified.)  For example, to sort by the fist column lexically ascending and then by the second column lexically descending, use ``dtk sort -s 0 -d 1``.  To sort by the first column numerically ascending and then by the second column lexically descending, use ``dtk sort 0 -sd 1``.

DTK ``sort`` uses the rules to precalculate keys for each row; this design choice makes it very fast at the cost of memory.  Each rule produces a new key for each row; once all sort keys are calculated, the entire data structure is sorted based on the ordering and comparison rules.

Valid macros are:

<dl>
<dt><code>time</code></dt><dd>convert the given column to a unix timestamp (using <a href="http://search.cpan.org/perldoc?Date::Parse">Date::Parse</a>) and force <code>-n</code>.</dd>
<dt><code>domain</code></dt><dd>the given column is split by dots, reversed, and sorted by <code>-s</code> (so "b.com" comes before "a.net").</dd>
<dt><code>mixed</code></dt><dd>the given column is broken into alternating groups of digit/nondigit parts and sorted by <code>-n</code>/<code>-s</code> respectively (so "a2z" comes before "a10a").</dd>
</dl>

For example, to sort by the first column if it contains human-readable times, pipe data to ``dtk sort 0:time``.

Here is a sort which takes a three-column input (two numbers and a string) and sorts it by the string (descending) followed by the sum of the two numbers (ascending):

```
$ perl -e 'my @str=qw(a b c); print rand()."\t".rand()."\t".$str[rand()*@str]."\n" for 1..30' \
  | dtk sort -sd '$v[2]' -na '$v[0] + $v[1]'
```

Here is a sort which produces tab-delimited ip/time pairs from the first ten lines of an apache access log, sorting by date and, where the dates match, by individual IP octets.

```
$ cat access_log | dtk parse apache_access ip,datetime | head -n 10 | dtk sort 1:time 0:mixed
```

Here is an example of sorting a list of domains using first the standard GNU ``sort`` utility (much like ``dtk sort -s 0``) and then using the ``domain`` macro:

```
$ cat hostlist | sort
a.example2.com
a.example.com
a.example.net
b.example2.com
b.example.com
b.example.net
c.example2.com
c.example2.net
```

```
$ cat hostlist | dtk sort 0:domain
a.example.com
b.example.com
a.example2.com
b.example2.com
c.example2.com
a.example.net
b.example.net
c.example2.net
```

## ``stats``

This module expects records containing a single numeric field and calculates statistics from them:

```
$ perl -e 'print rand()**2, "\n" for 1..100000' | dtk stats
count:    100000
min:      2.15895829436864e-10
max:      0.999996576152458
sum:      33383.415546156
mean:     0.33383415546156
variance: 0.0890887085401628
stddev:   0.298477316625841
skew:     0.638667363455869
```

## ``uc``

This module counts up the number of times it receives each whole record on STDIN.  It behaves much like ``... | sort | uniq -c | sort -n | head -n ...``, except that it is much faster (by building a hash table instead of sorting, but at the expense of memory) and supports easy ordering and limits.  Its only arguments are the number of lines after which output should stop (like ``head -n ...``) and either ``asc`` or ``desc`` to choose the sorting order (like ``sort -n`` or ``sort -nr``).  By default, ``dtk uc`` will sort the output by count ascending with no output limit.  The output is one line of output for every distinct line of input, prefixed by the number of times that line was seen and a tab.

```
$ perl -e 'print int(rand()**2*10), "\n" for 1..100000' | head -n 10
5
4
1
7
2
5
0
5
4
3

$ perl -e 'print int(rand()**2*10), "\n" for 1..100000' | dtk uc desc 5
31624   0
12971   1
10230   2
8539    3
7400    4
```

## Histogram modules

*Note:* You can read more about histograms on [Wikipedia](http://en.wikipedia.org/wiki/Histogram).

These tools are part of the DTK histogram suite.  They all take newline-terminated records containing tab-delimited fields on STDIN and use the following conventions.

Options on a specific input column are of the format ``v<n><p>=<v>``, where ``<n>`` is the input column (counting from zero), ``<p>`` is the property to set, and ``<v>`` is the new value for the property.  For example, to set the number of buckets for the second column to ``10``, use ``v1bn=10``.  The following options are available for any module in the histogram suite:

<dl>
<dt><code>v&lt;n&gt;t</code> </dt><dd>type; 'n' for numeric (group by numeric ranges), 's' for string (group each value separately)</dd>
<dt><code>v&lt;n&gt;bn</code></dt><dd>bucket number; the integer number of buckets to use</dd>
<dt><code>v&lt;n&gt;bs</code></dt><dd>bucket size; the width of each bucket</dd>
<dt><code>v&lt;n&gt;bd</code></dt><dd>bucket disable; set to 1 to disable bucketing; usually set automatically when needed</dd>
<dt><code>v&lt;n&gt;bo</code></dt><dd>bucket order; 'value' or 'count' (default: 'value')</dd>
<dt><code>v&lt;n&gt;cl</code></dt><dd>count low; the minimum count a bucket from this column must have to be included (default: no limit)</dd>
<dt><code>v&lt;n&gt;f</code> </dt><dd>format; takes sprintf or strftime formats (as specified by <code>v&lt;n&gt;ft</code>)</dd>
<dt><code>v&lt;n&gt;ft</code></dt><dd>format type; either <code>sprintf</code> or <code>strftime</code>; defaults to <code>sprintf</code> with <code>v&lt;n&gt;f=%.02f</code>; <code>v&lt;n&gt;ft=strftime</code> with no <code>v&lt;n&gt;f</code> defaults to <code>v&lt;n&gt;f="%F %T"</code></dd>
</dl>

Only one of ``bn``/``bs``/``bd`` can be set for each input column and can only be set for numeric (``v<n>t=n``) columns.

For example:
* break the first column into ten buckets: ``v0bn=10``
* buckets of width .2 for the second input column: ``v1bs=.2``
* first column is 20 formatted buckets, second column is string: ``v0bn=20 v0f=%.04f v1t=s``


### ``hist1``

This module produces single-axis histograms. The output shows the minimum and maximum value that went into each bucket (min..max), the number of values that went into the bucket, and the bar of the histogram. The bar is drawn with ``=`` and capped with ``-`` when the last character of the bar would have been at least half-filled.

For example, here's a single-axis histogram of the sum of rolling three six-sided dice:

```
$ perl -e 'print d6()+d6()+d6(),"\n" for 1..100000; sub d6{int rand(6)+1}' | dtk hist1 v0f=%d v0bs=1
 3..3    487 ==-
 4..4   1366 =======
 5..5   2871 ==============-
 6..6   4565 =======================
 7..7   7047 ====================================
 8..8   9766 ==================================================
 9..9  11482 ==========================================================-
10..10 12427 ===============================================================-
11..11 12650 =================================================================
12..12 11722 ============================================================
13..13  9532 ================================================-
14..14  6831 ===================================
15..15  4624 =======================-
16..16  2746 ==============
17..17  1448 =======
18..18   436 ==
```

Here's an example which unzips a bunch of log files, pulls out the response sizes (converting ``-`` to ``0`` via ``:numeric``), filters out the stray hits over 1kb (this server mostly does small queries), and then renders a single-axis histogram (``dtk hist1``) with 40 buckets (``v0bn=40``) and rendering the values as integers (``v0f=%d``).

```
$ zcat access_log.[123456789].gz \
  | dtk parse apache_access bytes:numeric \
  | dtk filter 'next unless $v[0] < 1024' \
  | dtk hist1 v0f=%d v0bn=40

   0..24   341466 ====================================================================================================
  26..50      521 
  51..75     1654 
  76..100     415 
 101..125    7214 ==
 126..148   80721 =======================-
 151..175    9126 ==-
 177..193    2439 -
 201..224     500 
 230..243     835 
 262..274      22 
 279..300      87 
 301..324     565 
 326..349      39 
 361..375     161 
 379..395    1015 
    ..          0 
 448..448       1 
 454..463      71 
    ..          0 
 523..526      41 
 530..547      70 
 559..566      69 
 581..581      33 
 612..624   13183 ===-
 632..634     117 
 655..676      98 
 693..701     390 
 710..716      99 
 732..749     104 
 754..761      42 
 779..786     265 
 803..814     183 
    ..          0 
    ..          0 
    ..          0 
 905..905       2 
    ..          0 
    ..          0 
1002..1002      4 
```

Above, you can see the minimum and maximum values which landed in each bucket (like ``454..463`` or ``1002..1002``), the number of values which fell into that bucket (like ``71`` or ``4``), and a bar showing the relative magnitude of the number of values in that bucket as compared to the others.  Buckets which had no values show no minimum or maximum value (merely <code>&nbsp;&nbsp;..&nbsp;&nbsp;</code>).

### ``hist2``

This module renders two-axis histograms. 2d buckets are formed by pairing each row bucket with each column bucket. All buckets are scaled relative to the size of the single largest bucket. Each bucket is drawn both with a color (black &rarr; white in 256-color mode, black &rarr; rainbow &rarr; white in 16-color mode) and a number (``00`` &rarr; ``99``) indicating the relative height of that bar.

Here is a two-axis histogram which compares the incoming user agents with the time of day for a single day of logs.  In it, all buckets are relative to the same scale, so it can be seen that while ``other`` user agents (curl, web crawlers, etc) send requests consistently, they do so at about 1/5 of what ``firefox`` user agents send at peak.  It can also be seen that users of this service seem to use it primarily between roughly 9:00am and 9:00pm, but not during the night.

```
$ zcat access_log.gz | dtk parse apache_access datetime:epoch,useragent:class | dtk hist2 v0ft=strftime v1t=s
```

![hist2 example 1](https://raw.github.com/synacorinc/dtk/master/doc-images/example_hist2_001.png)

Here is another example which shows the number of responses per four-hour block to non-``other`` user agents broken down by logarithmically-scaled response size:

```
$ zcat access_log.gz \
  | dtk parse apache_access datetime:epoch,bytes:numeric,useragent:class \
  | dtk filter 'next if $v[2] eq "other"; $v[1]&&=log($v[1])/log(10); @v=@v[0,1]' \
  | dtk hist2 v0ft=strftime v0bs=14400 v1bs=1
```

![hist2 example 2](https://raw.github.com/synacorinc/dtk/master/doc-images/example_hist2_002.png)

### ``hist2_bycol``

This module is identical to ``hist2`` except that buckets are scaled relative to the highest bucket in their respective column rather than to every bucket in the histogram.

Here is the first example from ``hist2`` but using ``hist2_bycol`` instead.  In it, because each column (each user agent class) is scaled separately, we can more clearly see the bursts from the ``other`` user agents around 3:00am, 10:00am, and (a smaller one around) midnight.

![hist2_bycol example 1](https://raw.github.com/synacorinc/dtk/master/doc-images/example_hist2bycol_001.png)

Here is the second example from ``hist2`` but again using ``hist2_bycol`` instead.  With each column scaled separately, we can more clearly see the shape of the distribution for each range of response sizes.

```
# zcat access_log.gz \
  | dtk parse apache_access datetime:epoch,bytes:numeric,useragent:class \
  | dtk filter 'next if $v[2] eq "other"; $v[1]&&=log($v[1])/log(10); @v=@v[0,1]' \
  | dtk hist2_bycol v0ft=strftime v0bs=14400 v1bs=1
```

![hist2_bycol example 2](https://raw.github.com/synacorinc/dtk/master/doc-images/example_hist2bycol_002.png)

### ``delta1``

This module is like ``hist1`` except that it takes a second column indicating the weight of the entry (including negative or non-integer weights). The histogram produced will have a zero line down the middle and will show, for each bucket, the relative total positive or negative value.

This example uses similar data to the first example for ``hist2`` and ``hist2_bycol``; it runs the data through a ``dtk filter`` command which gives requests having an ``other`` user agent a weight of ``-1`` (the control group) and all other requests a weight of ``1`` (the test group).  With this diagram, we pretend that the ``other`` data constitutes some baseline (in this case, "non-human interaction") and we compare it to the rest of the traffic (in this case, "human interaction"), which effectively highlights for us when human users are more likely (green) or less likely (red) to be interacting with the service than an automated user agent:

```
$ zcat access_log.gz \
  | dtk parse apache_access datetime:epoch,useragent:class \
  | dtk filter '$v[1] = $v[1] eq "other" ? -1 : 1' \
  | dtk delta1 v0ft=strftime v0bs=14400 v1f=%d
```

![delta1 example 1](https://raw.github.com/synacorinc/dtk/master/doc-images/example_delta1_001.png)
