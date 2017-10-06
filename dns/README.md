# Resources
- https://blog.cloudflare.com/the-story-of-a-little-dns-easter-egg/
- https://blog.cloudflare.com/how-we-made-our-dns-stack-3x-faster/
- https://blog.cloudflare.com/how-and-why-the-leap-second-affected-cloudflare-dns/
- https://blog.powerdns.com/2015/03/11/introducing-dnsdist-dns-abuse-and-dos-aware-query-distribution-for-optimal-performance/
- https://blog.powerdns.com/page/11/
- https://www.powerdns.com/platform.html

# Stuff
- PowerDNS comment:
```sh
PowerDNS with bind backend to support a stealth master setup for AXFR 's as well as the GEO backend for geo-targeted. This works great for me as it's all flat files. When incomplete AXFR's happen or bad data comes down from the masters I can just nuke the file then tell pdns_control to reload <zonename> and it will re-initialize the file from the master.
```
