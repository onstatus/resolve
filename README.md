# resolve.py

resolve.py  
A command line iterative DNS resolution testing program  
Author: Shumon Huque

A command line tool to perform iterative DNS resolution of a single
DNS name, type, and class. If either type or class or both are omitted, 
then a  default type of 'A' (IPv4 address record), and a default class 
of 'IN' (Internet class) are used.

Pre-requisites:  
- Python 2.7 or later, or Python 3
- [dnspython module](http://www.dnspython.org/) (included with most Linux/*BSD distributions)

```
resolve.py version 0.15

Usage: resolve.py [-dmtsnxez] <qname> [<qtype>] [<qclass>]
       resolve.py [-dmtsnxez] -b <batchfile>

     -d: print debugging output
     -m: do qname minimization
     -t: use TCP only
     -v: verbose - trace query & zone path
     -s: print summary statistics
     -n: resolve all non-glue NS addresses in referrals
     -x: workaround NXDOMAIN on empty non-terminals
     -e: don't use EDNS0 (default is EDNS0 with payload=1460)
     -z: set DNSSEC_OK flag (default is do not)
     -b <batchfile>: batch file mode

When using -b, <batchfile> contains one (space separated) query name, type,
class per line.
```

This program implements the normal iterative DNS resolution algorithm 
described in the DNS protocol specifications. Here's an example
invocation with the -v (verbose) switch to lookup the IPv6 address
of the server www.seas.upenn.edu:

```
$ ./resolve.py -v www.seas.upenn.edu. AAAA
>> Query: www.seas.upenn.edu. AAAA IN at zone .
>>        [Got Referral to zone: edu.]
>> Query: www.seas.upenn.edu. AAAA IN at zone edu.
>>        [Got Referral to zone: upenn.edu.]
>> Query: www.seas.upenn.edu. AAAA IN at zone upenn.edu.
www.seas.upenn.edu. 120 IN AAAA 2607:f470:8:64:5ea5::9
```

### Batch mode

If executing many different queries, then it is recommended to use
the batch mode (-b inputfile). This will cause the program to use its
cache of previously queried zones and nameserver records, increasing
performance, and reducing the possibility of responses being rate
limited by authoritative servers.

The format of the batch input file is a space-separated query-name,
query-type, and query-class per line. The type and class if omitted
default to 'A' and 'IN'.


### Query-name minimization mode

When invoked with the -m switch, this program uses a **query name 
minimization** algorithm that exposes only the needed query labels to 
authoritative servers as it traverses the DNS delegation hierarchy down 
to the target DNS zone. This is a more *privacy preserving* mode of DNS 
resolution, and efforts are underway to standardize this mode in the 
official DNS protocol specifications. There are a number of different 
ways a query name minimization algorithm could be implemented. I chose 
to implement the simplest one that starts with one non-root label at 
the root DNS servers, and successively prepends additional labels as 
it follows referrals and descends zones.

Here's an example run with qname minimization (-m) and the verbose (-v)
option, to resolve the amazon.com website:

```
$ ./resolve.py -vm www.amazon.com
>> Query: com. A IN at zone .
>>        [Got Referral to zone: com.]
>> Query: amazon.com. A IN at zone com.
>>        [Got Referral to zone: amazon.com.]
>> Query: www.amazon.com. A IN at zone amazon.com.
>>        [Got Referral to zone: www.amazon.com.]
>> Query: www.amazon.com. A IN at zone www.amazon.com.
www.amazon.com. 60 IN A 176.32.98.166
```

Some Content Delivery Networks (CDN) like Akamai and Cloudflare have 
problems with minimized query names, because they respond incorrectly
to intermediate query names with NXDOMAIN (response code 3). The correct
response should be NOERROR, AA-bit set, and an empty answer section. 
Invoking resolve.py with the -x switch implements a hack that works 
around this incorrect behavior by ignoring intermediate NXDOMAIN 
responses. The Cloudflare servers additionally appear to respond to 
some intermediate qnames with REFUSED.

This behavior of the Akamai and Cloudflare DNS servers was observed 
in January 2015. Hopefully they will get fixed before qname minimization 
is widely deployed.

An example resolution of www.upenn.edu (on Akamai):

```
$ ./resolve.py -vm www.upenn.edu. A
>> Query: edu. A IN at zone .
>>        [Got Referral to zone: edu.]
>> Query: upenn.edu. A IN at zone edu.
>>        [Got Referral to zone: upenn.edu.]
>> Query: www.upenn.edu. A IN at zone upenn.edu.
www.upenn.edu. 300 IN CNAME www.upenn.edu-dscg.edgesuite.net.
>> Query: net. A IN at zone .
>>        [Got Referral to zone: net.]
>> Query: edgesuite.net. A IN at zone net.
>>        [Got Referral to zone: edgesuite.net.]
>> Query: edu-dscg.edgesuite.net. A IN at zone edgesuite.net.
ERROR: NXDOMAIN: edu-dscg.edgesuite.net. not found
www.upenn.edu. 300 IN CNAME www.upenn.edu-dscg.edgesuite.net.
```

Repeating the query with -x (intermediate NXDOMAIN workaround) allows
the program to proceed to the final answer:

```
$ ./resolve.py -vmx www.upenn.edu
>> Query: edu. A IN at zone .
>>        [Got Referral to zone: edu.]
>> Query: upenn.edu. A IN at zone edu.
>>        [Got Referral to zone: upenn.edu.]
>> Query: www.upenn.edu. A IN at zone upenn.edu.
www.upenn.edu. 300 IN CNAME www.upenn.edu-dscg.edgesuite.net.
>> Query: net. A IN at zone .
>>        [Got Referral to zone: net.]
>> Query: edgesuite.net. A IN at zone net.
>>        [Got Referral to zone: edgesuite.net.]
>> Query: edu-dscg.edgesuite.net. A IN at zone edgesuite.net.
ERROR: NXDOMAIN: edu-dscg.edgesuite.net. not found
>> Query: upenn.edu-dscg.edgesuite.net. A IN at zone edgesuite.net.
ERROR: NXDOMAIN: upenn.edu-dscg.edgesuite.net. not found
>> Query: www.upenn.edu-dscg.edgesuite.net. A IN at zone edgesuite.net.
www.upenn.edu-dscg.edgesuite.net. 21600 IN CNAME a1165.dscg.akamai.net.
>> Query: akamai.net. A IN at zone net.
>>        [Got Referral to zone: akamai.net.]
>> Query: dscg.akamai.net. A IN at zone akamai.net.
>> Query: a1165.dscg.akamai.net. A IN at zone akamai.net.
>>        [Got Referral to zone: dscg.akamai.net.]
>> Query: a1165.dscg.akamai.net. A IN at zone dscg.akamai.net.
www.upenn.edu. 300 IN CNAME www.upenn.edu-dscg.edgesuite.net.
www.upenn.edu-dscg.edgesuite.net. 21600 IN CNAME a1165.dscg.akamai.net.
a1165.dscg.akamai.net. 20 IN A 23.62.6.59
a1165.dscg.akamai.net. 20 IN A 23.62.6.81
```

Resolving www.ietf.org (on Cloudflare) with the NXDOMAIN workaround
shows the following:

In this case, the first empty non-terminal, cdn.cloudflare.net returns
NXDOMAIN, the next one, org.cdn.cloudflare.net returns REFUSED, the
next one, ietf.org.cdn.cloudflare.net responds correctly (NOERROR, AA-bit,
empty answer), and the final name www.ietf.org.cdn.cloudflare.net produces
the answer records.

```
$ ./resolve.py -vmx www.ietf.org
>> Query: org. A IN at zone .
>>        [Got Referral to zone: org.]
>> Query: ietf.org. A IN at zone org.
>>        [Got Referral to zone: ietf.org.]
>> Query: www.ietf.org. A IN at zone ietf.org.
www.ietf.org. 1800 IN CNAME www.ietf.org.cdn.cloudflare.net.
>> Query: cloudflare.net. A IN at zone net.
>>        [Got Referral to zone: cloudflare.net.]
>> Query: cdn.cloudflare.net. A IN at zone cloudflare.net.
ERROR: NXDOMAIN: cdn.cloudflare.net. not found
>> Query: org.cdn.cloudflare.net. A IN at zone cloudflare.net.
WARNING: response REFUSED from 173.245.59.31
WARNING: response REFUSED from 2400:cb00:2049:1::adf5:3b1f
WARNING: response REFUSED from 198.41.222.131
WARNING: response REFUSED from 2400:cb00:2049:1::c629:de83
WARNING: response REFUSED from 198.41.222.31
WARNING: response REFUSED from 2400:cb00:2049:1::c629:de1f
WARNING: response REFUSED from 198.41.223.131
WARNING: response REFUSED from 2400:cb00:2049:1::c629:df83
WARNING: response REFUSED from 198.41.223.31
WARNING: response REFUSED from 2400:cb00:2049:1::c629:df1f
ERROR: Queries to all servers for zone cloudflare.net. failed.
>> Query: ietf.org.cdn.cloudflare.net. A IN at zone cloudflare.net.
>> Query: www.ietf.org.cdn.cloudflare.net. A IN at zone cloudflare.net.
www.ietf.org. 1800 IN CNAME www.ietf.org.cdn.cloudflare.net.
www.ietf.org.cdn.cloudflare.net. 300 IN A 104.20.0.85
www.ietf.org.cdn.cloudflare.net. 300 IN A 104.20.1.85
```
