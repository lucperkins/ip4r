
IP4R  - IPv4/v6 and IPv4/v6 range index type for PostgreSQL
===========================================================

CHANGES in version 2.4:
=======================

 * Add new cidr_split functions to decompose an arbitrary range into
   a set of CIDR blocks.

 * Add casts to and from bit and bytea types.

 * Support new hash functions for hash partitioning.

 * Support window function RANGE offsets in pg11+, including a special
   case where negative offset values are treated as CIDR prefix lengths
   (so RANGE BETWEEN -16 PRECEDING AND -16 FOLLOWING includes in the
   window frame all rows in the same /16 as the current row).

 * Fix a historical oversight with the hash function for iprange, in which
   it would return different values for ip6 cidr ranges than the ip6r hash
   function. This was not a bug in that no queries would give wrong
   answers, but it would prevent any future reorganization of the opclass
   to handle cross-type comparisons. Since allowing hash partitioning has
   the practical effect of casting the hash function in stone (far more so
   than hash indexes do), best to fix this now.

   The result of the function iprangehash(iprange) is NOT changed by this
   update, on the assumption that it might have been in use for
   inheritance partitioning or other explicit uses. Instead, a new hash
   function is added for the opclass to use.

   NOTE: This version requires any hash indexes on iprange columns to be
   rebuilt. An ALTER EXTENSION ... UPDATE command will refuse to run if it
   detects any such indexes (it will report the offending indexes in INFO
   messages to the client). You can automatically drop or recreate
   affected indexes by performing one of these commands before the ALTER
   EXTENSION:

     SET ip4r.update_indexes = 'drop';
     SET ip4r.update_indexes = 'rebuild';

   (on pg 9.1 you will have to add 'ip4r' to custom_variable_classes
   in postgresql.conf and restart before doing this)

   The value 'drop' will drop any iprange hash indexes and complete the
   upgrade. The DROP commands will be done with RESTRICT: if there are any
   additional dependencies on the indexes in question, the upgrade will
   not proceed; these will have to be dealt with manually. If any indexes
   were dropped the upgrade will leave behind a new table named
   ip4r_update_to_2_4.update_indexes containing the affected table and
   index information, including the CREATE INDEX command as obtained by
   pg_get_indexdef(). This table is not part of the extension and can be
   dropped when the data is no longer needed. The value 'rebuild' will
   cause the update script to perform the needed CREATE INDEX commands
   itself (which of course may take some time).

CHANGES in version 2.3:
=======================

 * Fix build for changes in PostgreSQL 11.

 * Fix long-standing bug in binary-mode I/O of iprange type
   (other types were not affected).

CHANGES in version 2.2:
=======================

 * Updating to this version should fix the pg_dump issue that affects
   certain older installs that have been updated to 2.1, where a
   poorly written catalog update to fix old function signatures broke
   pg_dump.

CHANGES in version 2.1:
=======================

 * Index-only scans are now supported (on pg 9.5 or later)

 * Downward casts from ipaddress to ip4/ip6, and iprange to
   ip4r/ip6r, are now allowed as assignment casts as well as
   explicit casts.

 * parallel-safe is set on all functions in pg 9.6 or later.

 * Support for pre-9.1 versions of postgres (back to 8.4), and
   non-extension builds on any version, is maintained in this release
   (and some glaring bugs fixed), but should be considered
   deprecated. Future releases will likely abandon support for
   non-extension packaging.

 * Vestigial (and apparently non-functional) support for versions
   prior to 8.4 is removed.

CHANGES in version 2.0:
=======================

 * 9.1+ extension packaging mechanism is the default for this version
   (use NO_EXTENSION=1 to build on pre-9.1 or without packaging)

 * New types for ip6, ip6r, ipaddress, iprange

 * ip4 input no longer accepts spurious leading whitespace

UPGRADING
=========

If upgrading from ip4r-2.0 installed outside the extension mechanism, use:

CREATE EXTENSION ip4r FROM "unpackaged2.0";

If upgrading from ip4r-1.x, use:

CREATE EXTENSION ip4r FROM unpackaged1;


RATIONALE
=========

While PostgreSQL already has builtin types 'inet' and 'cidr', the
authors of this module found that they had a number of requirements
that were not addressed by the builtin type.

Firstly and most importantly, the builtin types have no good support
for index lookups of the form (column >>= parameter), i.e. where you
have a table of IP address ranges and wish to find which ones include
a given IP address.  This requires an rtree or gist index to do
efficiently, and also requires a way to represent IP address ranges
that do not fall precisely on CIDR boundaries.

(While newer versions of PostgreSQL do now have support for gist
indexes on the inet type, the performance is very poor compared to
this module.)

Secondly, the builtin inet/cidr are somewhat overloaded with
semantics, with inet combining two distinct concepts (a netblock, and
a specific IP within that netblock). Furthermore, they are variable
length types (to support ipv6) with non-trivial overheads, and the
authors (whose applications mainly deal in large volumes of single
IPv4 addresses) wanted a more lightweight representation.

IP4R therefore supports six distinct data types:

  ip4   - a single IPv4 address
  ip4r  - an arbitrary range of IPv4 addresses
  ip6   - a single IPv6 address
  ip6r  - an arbitrary range of IPv6 addresses
  ipaddress  - a single IPv4 or IPv6 address
  iprange    - an arbitrary range of IPv4 or IPv6 addresses

Simple usage examples:

CREATE TABLE ipranges (range ip4r primary key, description text not null);
CREATE INDEX ipranges_range_idx ON ipranges USING gist (range);
INSERT INTO ipranges VALUES ('10.0.0.0/8','rfc1918 block 1');
INSERT INTO ipranges VALUES ('172.16.0.0/12','rfc1918 block 2');
INSERT INTO ipranges VALUES ('192.168.0.0/16','rfc1918 block 3');
INSERT INTO ipranges VALUES ('0.0.0.0/1','classical class A space');
INSERT INTO ipranges VALUES ('10.0.1.10-10.0.1.20','my internal network');
INSERT INTO ipranges VALUES ('127.0.0.1','localhost');

CREATE TABLE access_log (id serial primary key, ip ip4 not null);
CREATE INDEX access_log_ip_idx ON access_log (ip);
INSERT INTO access_log(ip) VALUES ('10.0.1.15');
INSERT INTO access_log(ip) VALUES ('24.1.2.3');
INSERT INTO access_log(ip) VALUES ('192.168.10.20');
INSERT INTO access_log(ip) VALUES ('127.0.0.1');

-- find all accesses from 10.0.0.0/8
SELECT * FROM access_log WHERE ip BETWEEN '10.0.0.0' AND '10.255.255.255';

-- find all applicable descriptions for all entry in the access log
-- returns multiple rows for each entry if there are overlapping ranges
SELECT id,ip,range,description FROM access_log, ipranges WHERE ip <<= range;

-- find only the most specific description for all IPs in the access log
SELECT DISTINCT ON (ip) ip,range,description
  FROM access_log, ipranges
 WHERE ip <<= range
 ORDER BY ip, @ range;


INSTALLATION
============

ip4r can be installed via the pgxs mechanism (which is now the default).
Unpack the distribution and do:

make
make install

(as with PostgreSQL itself, this requires GNU Make. The second command
will usually need to be run as root.)

PRE-9.1 SYSTEMS
===============

On versions before 9.1, or to build without extension packaging, use:

make NO_EXTENSION=1
make NO_EXTENSION=1 install

and execute the ip4r.sql file manually in the same way as for contrib
modules (see the postgres manual).


USAGE
=====

Types "ip4", "ip6", "ipaddress"
-------------------------------

"ip4" accepts input in the form 'nnn.nnn.nnn.nnn' in decimal base only
(no hex, octal, etc.).  An ip4 value is a single IP address, and is
stored as a 32-bit unsigned integer.

"ip6" accepts input in the standard hexadecimal representation for
IPv6 addresses, e.g. '2001:1234:aa55::2323'. "Mixed" format input
(using an IPv4 dotted-decimal for the last two words) is accepted. An
ip6 value is a single IP address, and is stored as two 64-bit values
for convenience. Output is represented according to the specification
in RFC 5952 (including output in mixed format for v4-mapped addresses).

"ipaddress" accepts any input which is valid for either ip4 or ip6. An
ipaddress value is a single IP address, either v4 or v6. The v4 and v6
ranges are treated as disjoint - all v4 addresses are considered lower
than all v6 addresses, and '1.2.3.4' and '::ffff:1.2.3.4' are not equal.

"ipX" will be used below to represent any of the above three types.

The following type conversions are supported:

  Source type   | Dest type  |  Form
----------------|------------|-------------------------------------------------
  ipX           |  text      |  text(ipX)  or  ipX::text  (explicit)
  text          |  ipX       |  ipX(text)  or  text::ipX  (explicit)
  ipX           |  cidr      |  cidr(ipX)  or  ipX::cidr  (assignment)
  inet          |  ipX       |  ipX(inet)  or  inet::ipX  (assignment)
  ipX           |  numeric   |  to_numeric(ipX) or  ipX::numeric (explicit)
  numeric       |  ipX       |  ipX(numeric)    or  bigint::ipX  (explicit)
  ip4           |  bigint    |  to_bigint(ip4)  or  ip4::bigint  (explicit)
  bigint        |  ip4       |  ip4(bigint)     or  bigint::ip4  (explicit)
  ip4           |  float8    |  to_double(ip4)  or  ip4::float8  (explicit)
  float8        |  ip4       |  ip4(float8)     or  float8::ip4  (explicit)
  ipX           |  varbit    |  to_bit(ipX)     or  ipX::varbit  (explicit)
  bit(32)       |  ip4       |  ip4(bit)        or  bit::ip4     (explicit)
  bit(128)      |  ip6       |  ip6(bit)        or  bit::ip6     (explicit)
  varbit        |  ipX       |  ipX(varbit)     or  varbit::ipX  (explicit)
  ipX           |  bytea     |  to_bytea(ipX)   or  ipX::bytea   (explicit)
  bytea         |  ipX       |  ipX(bytea)      or  bytea::ipX   (explicit)
  ipX           |  ipXr      |  ipXr(ipX)  or  ipX::ipXr  (implicit)
  ip4           |  ipaddress |  ipaddress(ip4)  or  ip4::ipaddress (implicit)
  ip6           |  ipaddress |  ipaddress(ip6)  or  ip6::ipaddress (implicit)
  ipaddress     |  ip4       |  ip4(ipaddress)  or  ipaddress::ip4 (assignment)
  ipaddress     |  ip6       |  ip6(ipaddress)  or  ipaddress::ip6 (assignment)

The conversions from bigint and float8 are available only for ip4, and
accept values which are exact integers in the range 0 .. 2^32-1, which
are converted to IPs in the range 0.0.0.0 - 255.255.255.255 in the
obvious way. This is useful for conversions from applications which
store IPs in numeric form, as is often done for performance in certain
other databases.

Conversions to and from the 'numeric' type are available for all
formats with the obvious behaviour.

The conversion to cidr always results in a /32 (for v4) or /128 (for v6).
The conversion from inet ignores any prefix length and just takes the
specific IP address.

An ipX value implicitly converts to either the corresponding range
type (ip4 -> ip4r, ip6 -> ip6r), or to the iprange type, producing a
range containing only the single IP address.

ipX supports the following operators with the conventional meanings:
=, <>, <, >, <=, >=, and supports ORDER BY and btree indexes in the
obvious fashion. However, the planner does not understand how to
transform a query of the form

  WHERE ipcolumn <<= value

into a btree range scan (it does this transformation for the builtin
inet type using a function which is not extensible by plugins). As a
workaround, use the following form instead:

  WHERE ipcolumn BETWEEN lower(value) AND upper(value)

which will use a btree range scan.

ipX supports the following additional operators and functions:

 family(ipX) returns integer
 | returns the value 4 or 6 depending on address family

 ip4_netmask(integer) returns ip4
 | returns an ip4 value that represents a netmask for a prefix length

 ip6_netmask(integer) returns ip6
 | returns an ip6 value that represents a netmask for a prefix length

 ipX_net_lower(ipX, integer) returns ipX
 | returns the lowest address in the cidr block of the specified prefix
 | length, containing the specified IP
 | equivalent to: network(set_masklen(cidr(ipX),integer))

 ipX_net_upper(ipX, integer) returns ipX
 | returns the highest address in the cidr block of the specified prefix
 | length, containing the specified IP
 | equivalent to: broadcast(set_masklen(cidr(ip4),integer))

  Operator        | Description
------------------|--------------------------------------------------------
 ipX + integer    | add the given integer to the IP 
 ipX - integer    | subtract the given integer from the IP 
 ipX + bigint     | add the given integer to the IP 
 ipX - bigint     | subtract the given integer from the IP 
 ipX + numeric    | add the given integer to the IP 
 ipX - numeric    | subtract the given integer from the IP 
 ipX - ipX        | (returns bigint or numeric) difference between two IPs
 ipX & ipX        | bitwise-AND the two values
 ipX | ipX        | bitwise-OR the two values
 ipX # ipX        | bitwise-XOR the two values
 ~ ipX            | bitwise-NOT the value

Arithmetic on ip4 values does not wrap below 0.0.0.0 or above
255.255.255.255 - attempting to go beyond these limits raises an
error.

More complex arithmetic on IP addresses can be performed by converting
the IPs to numeric first; the above are only intended to cover the
common cases without requiring casts.


Types "ip4r", "ip6r", "iprange"
-------------------------------

An "ip4r" value denotes a single range of one or more IPv4 addresses,
for example '192.0.2.100-192.0.2.200'. Arbitrary ranges are allowed,
though input can also be in the form of CIDR netblocks, e.g.
'192.0.2.0/24' is equivalent to '192.0.2.0-192.0.2.255'. A single
value such as '192.0.2.25' represents a range containing only that
value.

An "ip6r" value denotes a single range of one or more IPv6 addresses,
for example '2001::1234-2001::2000:0000'. Arbitrary ranges are
allowed, though input can also be in the form of CIDR netblocks, e.g.
'2001::/112' is equivalent to '2001::-2001::ffff'. A single value such
as '2001::1234' represents a range containing only that value. Output
formatting is as specified in RFC 5952.

An "iprange" value denotes either an IPv4 range or an IPv6 range, or
the special value '-' which includes all of both IPv4 and IPv6 space.
Mixing of address families is not otherwise supported.

For all of the above types, values are displayed in CIDR form if they
represent a CIDR range, otherwise in range form.

Currently, abbreviated CIDR forms for IPv4 are not accepted at all,
i.e. all octets must be supplied. For IPv6, words may only be omitted
from the address as permitted by the zero-compression rules of RFC 5952.

"ipXr" will be used below to represent any one of the above three types.

An ipXr value can be constructed from two IPs explicitly using the
function ipXr(ipX,ipX). The ends of the range can be specified in
either order.

An ipXr value can be constructed from an IP and a prefix length
using the / operator (see below). For backward compatibility, the
function names ipXr_net_prefix and ipXr_net_mask are still accepted
for this operator.

ipXr supports the following type conversions:

  Source type   | Dest type |  Form
----------------|-----------|----------------------------------------------
  ipX           |  ipXr     |  ipXr(ipX)  or  ipX::ipXr  (implicit)
  ipXr          |  text     |  text(ipXr) or  ipXr::text (explicit)
  text          |  ipXr     |  ipXr(text) or  text::ipXr (explicit)
  ipXr          |  cidr     |  cidr(ipXr) or  ipXr::cidr (explicit)
  cidr          |  ipXr     |  ipXr(cidr) or  cidr::ipXr (assignment)
  ipXr          |  varbit   |  to_bit(ipXr) or ipXr::varbit  (explicit)
  varbit        |  ip4r     |  ip4r(varbit) or varbit::ip4r  (explicit)
  varbit        |  ip6r     |  ip6r(varbit) or varbit::ip6r  (explicit)

The conversion cidr(ipXr) returns NULL if the ipXr value does not
represent a valid CIDR range.

In addition, type conversions between ip4r, ip6r and iprange are permitted
in all valid combinations.

ipXr supports the following functions:

  family(ipXr) returns integer
  |  returns 4 or 6 according to address family, or NULL for '-'::iprange

  is_cidr(ipXr) returns boolean
  |  returns TRUE if the ipXr value is a valid CIDR range

  lower(ipXr) returns ipX
  |  returns the lower end of the ipXr range, as an ipX value

  upper(ipXr) returns ipX
  |  returns the upper end of the ipXr range, as an ipX value

  cidr_split(ipXr) returns setof ipXr
  |  splits the range up into separate CIDR blocks, and returns each one
  |  as a separate row

ipXr supports the following operators:

  Operator        | Description
------------------|--------------------------------------------------------
  a = b           | exact equality
  a <> b          | exact inequality
  a < b           | note [1]
  a <= b          | note [1]
  a > b           | note [1]
  a >= b          | note [1]
  a >>= b         | a contains b or is equal to b
  a >> b          | a strictly contains b
  a <<= b         | a is contained in b or is equal to b
  a << b          | a is strictly contained in b
  a && b          | a and b overlap
  @ a             | approximate size of a (returns double)
  @@ a            | exact size of a (returns numeric)
  a / n           | construct CIDR range from address a length n
  a / b           | construct CIDR range from address a netmask b

[1]: the operators <, <=, >, >= implement an ordering for the purposes of
btree indexes, DISTINCT and ORDER BY; the ordering is not necessarily
useful for applications. The ordering used is a lexicographic ordering
of (lower,upper).

For testing whether an ipXr range contains a specified single ip, use the
>>= operator, i.e.  ipXr >>= ipX.  The implicit conversion from ipX to ipXr
handles this case.


ipXr Indexes
------------

ipXr values can be indexed in several ways.

A conventional btree index on ipXr values will work for the purposes of
unique/primary key constraints, ordering, and equality lookups (i.e.
WHERE column = value). Btree indexes are created in the usual way and
are the default index type.

However, ipXr's utility comes from its ability to use gist indexes to
support the following lookup types:

  WHERE column >>= value      (or >>)
  WHERE column <<= value      (or <<)
  WHERE column && value

These lookups require a GiST index. This can be created as follows:

CREATE INDEX indexname ON tablename USING gist (column);

It is also possible to create a functional ip4r index over a column of
'cidr' type as follows:

CREATE INDEX indexname ON tablename USING gist (iprange(cidrcolumn));

(ip4r(column) or ip6r(column) can also be used if the column is constrained
to contain only values of the specified address family)

This can then be used for queries of the form:

  WHERE iprange(cidrcolumn) >>= value    (or >>, <<=, && etc)

One advantage of this method is that the ip4r type can be dropped and
recreated without losing data. This is useful for accelerating queries
on an existing table designed without ip4r in mind.

Another idiom sometimes seen for representation of ranges of IP
addresses is for applications to create two integer columns, and do
range queries of the form:

  WHERE value BETWEEN column1 and column2

This is an attempt to get some use out of a btree index, but it performs
poorly in most cases. This can also be converted to use a functional ip4r
index as follows:

CREATE INDEX indexname ON tablename
   USING gist (ip4r(column1::ip4,column2::ip4));

and then doing queries of the form:

  WHERE ip4r(column1::ip4,column2::ip4) >>= value

This method is not usually practical for IPv6.

A common requirement is to get the longest-prefix (most specific)
match to an IP address from a table of ranges or CIDR prefixes.
This can usually be best achieved using ORDER BY @ column,
for example:

SELECT * FROM tablename
 WHERE column >>= value
 ORDER BY @ column
 LIMIT 1

The use of @ column (approximate size) is sufficient if the values are
IPv4 ranges or are always CIDR prefixes. If arbitrary IPv6 ranges are
present, then overlapping ranges with small size differences might
compare equal; in this case use ORDER BY @@ column.

When looking up multiple IPs, one can do queries of the following
form:

SELECT DISTINCT ON (ips.ip) ips.ip, ranges.range
  FROM ips, ranges
 WHERE ranges.range >>= ips.ip
 ORDER BY ips.ip, @ ranges.range



AUTHORS
=======

this code by andrew@tao11.riddles.org.uk Oct 2004 - 2018
derived from 'ipr' by Steve Atkins <steve@blighty.com> August 2003
derived from the 'seg' type distributed with PostgreSQL.

Distributed under the same terms as PostgreSQL itself.

Currently maintained at:
  http://github.com/RhodiumToad/ip4r

