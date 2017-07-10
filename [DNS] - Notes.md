- DNS Architecture:
	- Master-Slave
	- Multi-masters
	- Stealth
- Anycast DNS
- DNSSEC
- Authoritative Name Server using BIND9/NSD
- Ref:
	- DNS and BIND: chapter 11 - Security
	- Building Internet Firewall: chapter 4 - Firewall Architecture
	
---

- CDN and DNS Solution comparison:
	- Akamai: 
		- Conventional approach: DNS-based server selection - redirects DNS look up from original server to content server (config CNAME on DNS Authoritative NS of original server). Client should use ISP's resolver for better routing rate. -> EDNS extention for external DNS resolver usage.
		- Fast DNS: Cloud-based DNS solution, using IP Anycast -> digging deeper later (ISP isolation, both Google DNS and Open DNS are OK)
		
	- Cloudflare:
		- Using IP Anycast for GBLB
		
- References:
	- https://anuragbhatia.com/2014/03/networking/different-cdn-technologies-dns-vs-anycast-routing/
	- https://github.com/thaihust/docs/blob/master/[DNS]%20Content%20Retrieval%20using%20Cloud-based%20DNS.pdf
	- https://github.com/thaihust/docs/blob/master/[Paper]%20DNS-based%20server%20selection.pdf
	- https://github.com/thaihust/docs/blob/master/[DNS-Akamai]%20akamai-designing-dns-for-availability-and-resilience-against-ddos-attacks.pdf
	- https://github.com/thaihust/docs/blob/master/[Akamai]%20Drafting%20behind%20Akamai.pdf
	- http://sci-hub.bz/10.1109/MIC.2002.1036038
	- https://blog.kloud.com.au/2017/05/01/akamai-cloud-based-dns-black-magic/
	- [EDNS1](https://community.akamai.com/docs/DOC-4219)
	- [EDNS2](https://blogs.akamai.com/2013/03/intelligent-user-mapping-in-the-cloud.html)
	- [Cloudflare Solution](https://tech.co/cloud-based-application-delivery-whos-leading-way-2015-04)
	- [Cloudflare Solution](https://www.cloudflare.com/argo/)
