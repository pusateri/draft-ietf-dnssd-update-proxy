---
title: DNS Update Proxy for mDNS
docname: draft-pusateri-dnssd-update-proxy-00
date: 2019
ipr: trust200902
area: Internet
wg: DNSSD Working Group
kw: Internet-Draft
cat: std

coding: utf-8
pi:
  - toc
  - symrefs
  - sortrefs

author:
  -
    ins: T. Pusateri
    name: Tom Pusateri
    org: Unaffiliated
    street: ""
    city: Raleigh
    code: NC 27608
    country: USA
    phone: +1 (919) 867-1330
    email: pusateri@bangj.com

--- abstract

This document describes a method to dynamically map multicast DNS announcements into the unicast DNS namespace for use by service discovery clients. It does not define any new protocols but uses existing DNS protocols in new ways. This solves existing problems with service discovery across multiple IP subnets in a simple, yet efficient, manner.

--- middle

# Introduction

Multicast DNS is used today for link-local service discovery. While this has worked reasonably well on the local link, current deployment reveals two problems. First, mDNS wasn't designed to traverse across multi-subnet campus networks. Second, IP multicast doesn't work across all link types and can be problematic on 802.11 Wifi networks. Therefore, a solution is desired to contain legacy multicast DNS service discovery and transition to a unicast DNS service discovery model. By mapping the current mDNS discovered services into regular authoritative unicast DNS servers, clients from any IP subnet can make unicast queries through normal unicast DNS resolvers.

There are many ways to map services discovered using multicast DNS into the unicast namespace. This document describes a way to do the mapping using a proxy that sends DNS Update messages {{!RFC2136}} directly to an authoritative unicast DNS server. While it is possible for each host providing a service to send it's own DNS Update, key management has prevented widespread deployment of DNS Updates across a domain. By having a limited number of proxies sitting on one or more IP subnets, it is possible to provide secure DNS updates at a manageable scale. Future work to automate secure DNS Updates on a larger scale is needed.

This document will explain how services on each .local domain will be mapped into the unicast DNS namespace and how unicast clients will discover these services. It is important to note that no changes are required in either the clients, DNS authoritative servers, or DNS resolver infrastructure. In addition, while the Update Proxy is a new logical concept, it requires no new protocols to be defined and can be built using existing DNS libraries.

An Update proxy is an ideal service to run on routers and/or switches to map local services into a larger network infrastructure.

# Requirements Language

{::boilerplate bcp14+}

# DNS subdomain model

Each .local domain which logically maps to an IP subnet is modeled as a separate subdomain in the unicast DNS hierarchy. Each of these subdomains must be browsable (respond to PTR queries for b._dns-sd._udp.&lt;subdomain&gt;.&lt;domain&gt;.). See Section 11 of {{!RFC6763}} for more details about browsing. In the context of the Update proxy, these subdomains are typically special use subdomains for mDNS mappings.

## Subdomain naming {#subdomain}

The browseable subdomain label is prepended to the domain name and separated by a period. See {{?RFC7719}} for more information on subdomains and labels. It is not important that the label be human readable or have organizational significance. End users will not be interacting with these labels. The main requirement is that they be unique within the domain for each IP subnet. Subdomain labels can be obtained by the proxy in several ways. The following methods should be attempted in order to assure consistency among redundant proxies:

1. address-derived domain enumeration through local resolver

    The proxy issues a PTR query for the registration or browse domains based on the IP subnet. Separate queries are performed for IPv4 and IPv6 on the same link since they are different IP subnets. Since the Update proxy will be registering services with DNS Update, it should begin querying for registration domains and fallback to browse domains if no registration domains are configured.

    As an example, suppose a proxy was connected to IPv4 subnet 203.0.113.0/24. In order to determine if there was a subdomain name for this subnet, the base domain name to query would be derived as 0.113.0.203.in-addr.arpa. The proxy would issue a PTR query for the following names in order to find the subdomain for the IP subnet:

    `dr._dns-sd._udp.0.113.0.203.in-addr.arpa.`
    
    `r._dns-sd._udp.0.113.0.203.in-addr.arpa.`
    
    `db._dns-sd._udp.0.113.0.203.in-addr.arpa.`
    
    `b._dns-sd._udp.0.113.0.203.in-addr.arpa.`
    
    `lb._dns-sd._udp.0.113.0.203.in-addr.arpa.`

    The first response with an answer should be the subdomain name including the domain name for the network and further queries through this list are not needed. If multiple answers are returned in the same response, any one of the answers can be used but the proxy should only use a single subdomain name for the IP subnet.

    The Update proxy should periodically rediscover the subdomain name at approximately 5 minute intervals for each IP subnet adding appropriate random jitter across IP subnets so as to prevent synchronization.

2. proxy local configuration override

    If no answer is returned, the proxy may have local configuration containing a subdomain name for the network. If so, this subdomain should be used.

3. algorithmic subdomain label generation

    If no local configuration is present for the IP subnet, the proxy may generate a unique label and use that for the subdomain by appending a common domain name. One such algorithm is to take the network form of an IPv4 subnet without a prefix length (host portion all zeros) and convert it to a hexadecimal string. This will give a 8 character unique string to use as a subdomain label. For the example above, this label would be cb007100.

## Domain name discovery

The base domain name to use for each subdomain also has to be discovered on a per IP subnet basis. In most cases, the domain name will be the same for all IP subnets because they are all contained in a single administrative domain. However, this is not required and a proxy administrator may need to span multiple administrative boundaries requiring different domain names on different IP subnets (and therefore, subdomains).

There is not a direct query to discover a separate domain name but the domain name is included with the subdomain in the response to the PTR query above in {{subdomain}}. If the PTR query returns an empty response, then the domain name can be obtained from local proxy configuration and if no domain name is specified there, the default domain for the host should be used.

## Client service discovery

Fortunately, clients performing service discovery require no changes in order to work with the Update proxy. Existing clients already support wide-area bonjour which specifies how to query search domains and subdomains for services. See section 11 of {{!RFC6763}}.

However, in order for clients to discover the subdomain for each IP subnet, the subdomain MUST be browseable and a browse record for the domain must enumerate all of the subdomains. If the domain records do not exist, the Update proxy MUST create them in the domain and MUST ensure each subdomain is browseable.

In the future, authoritative unicast DNS servers may add support for DNS Push Notifications {{?I-D.ietf-dnssd-push}} which would allow clients to maintain long lived subscriptions to services. Clients may also wish to add support for this feature to provide an efficient alternative to polling.

# Update proxy behavior

Since no new protocols are defined, this document mostly describes the expected behavior of the Update proxy and how it uses existing protocols to achieve multi IP subnet service discovery. The behavior is mostly intuitive but is described to ensure compatibility and completeness.

## mDNS service announcements

The Update proxy should listen to mDNS service announcements (responses) on all interfaces it is proxying for. Multiple Update proxies can be active on the same IP subnet at the same time. See {{!RFC6762}} for more information on multicast DNS.

## Service caching and refresh

As specified in Section 8.3 of {{!RFC6762}}, service announcements are sent multiple times for redundancy. However, there is no need to send duplicate Update messages to the authoritative unicast DNS server. Therefore, the Update proxy should cache service announcements and only send DNS Update messages when needed.

As described in Section 8.4 of {{!RFC6762}}, a host may send "goodbye" announcements by setting the TTL to 0. In this case, the record MUST be removed from the cache or otherwise marked as expired and a DNS Update should be sent to the authoritative unicast DNS server removing the record.

The Update proxy MUST also remove/expire old cache entries and remove the records from the authoritative unicast DNS server when the cache-flush bit is set on new announcements as described in Section 10.2 of {{!RFC6762}}.

A host providing a service may automatically refresh the TTL in the announcement from time to time keeping the service valid based on subsequent multicast queries it receives. However, if no mDNS clients are requesting the particular service for the length of the TTL value, the service announcement could timeout naturally. In order to keep accurate information regarding all of the services on the IP subnet, the Update proxy SHOULD send a unicast PTR query for the service name directly to the host announcing the service. This query should be sent at a random time between 5 and 10 seconds before the TTL value indicates the announcement will expire.

As described in Section 11 of {{!RFC6762}}, the Update proxy should use an IP source address of the IP subnet of the interface it is transmitting over and that is on the same IP subnet as the service provider. It is also permissible to use a link-local IP address in the IPv6 case as long as the service itself is available on an IPv6 address that is reachable from outside the local link.

In order for the Update proxy to discover as many services available on each IP subnet as possible, it should periodically send a PTR multicast query for `_services._dns-sd._udp.local` on each subnet. The unicast response bit SHOULD be set in the query in order to force unicast responses to the Update proxy. As PTR responses are received, The Update proxy can then send Service Instance Enumeration PTR queries (also with the unicast response bit set) for each service.

This was not the intended behavior of mDNS since local clients would just ask dynamically when they needed to know all of the providers of a service name but keeping this information up to date in the authoritative server provides benefits to remote clients such as faster response times and ability to use DNSSEC validation that were not previously possible with multicast DNS. These benefits are provided at the additional cost of a slight increase in network activity and processing time by the hosts announcing services. However, if the Update proxy uses unicast to query the service providers directly, other clients are not affected by these refresh queries and do not have to turn their radios on for queries/responses that they have no interest in.

## mDNS probing

While Section 8.2 of {{!RFC6762}} recommends all potential answers be included in mDNS probe queries, because these records haven't gone through conflict resolution, they should not be regarded as announcements of services. Therefore, an Update proxy MUST NOT rely on information in any section of DNS query messages.

## Link-local addressing

In the IPv6 case, the source address of the announcements is a link-local IPv6 address that will probably be different than the IP subnet that the service is being provided on. However, it is certainly possible that link-local addressing is used with IPv4 as well. This is not as common but exists in a zero-conf environment where no IPv4 addresses are assigned via DHCP or statically and the hosts revert to link-local IPv4 addresses (169.254/16), see {{!RFC3927}}.

If the service SRV target resolves to only a link-local address, then the service is not eligible to be advertised outside of the link and shouldn't be sent to the authoritative unicast DNS server by the Update proxy.

In general, the Update proxy needs to ensure that the service is reachable outside of the link it is announced on before sending an Update to the authoritative server for the subdomain.

## IPv6 and IPv4 on same link

Announced services may be available on IPv4, IPv6, or both on the same link. If both IPv4 `A` records {{!RFC1035}} and IPv6 `AAAA` records {{!RFC3596}} are published for an SRV target {{!RFC2782}} name, the administrator should provide the service over both protocols.

In some cases, this won't be possible. This will not incur any extra delays if clients attempt connections over both IPv4 and IPv6 protocols simultaneously but if one protocol is preferred over another, delays may occur.

## multiple logical IP subnets

Multiple IP subnets on the same link is just a more general case of IPv4 and IPv6 on the same link. When multiple IP subnets exist for the same protocol on the same link, they appear as separate interfaces to the Update proxy and require a separate subdomain name just as IPv4 and IPv6 do.

This is required for a client on one logical IP subnet of an interface to communicate with a service provided by a host on a different IP subnet of the same link.

If a SRV target resolves to addresses on multiple logical IP subnets of the same interface, the service can be included in multiple subdomains on the appropriate server(s) for those subdomains.

## Proxy redundancy

Providing redundant Update proxies for the same IP subnet can be easily achieved using the DNS Update protocol. None of the redundant proxies needs to be aware of any of the other redundant proxies on an IP subnet.

Alternatives for ways to format DNS Update messages are defined below in {{prereq}} as to possible uses of the Prerequisite section for use with redundant Update proxies.

## Service filtering and translation

In the process of registering services with an authoritative unicast DNS server, the proxy can perform filtering and translation on the dynamically discovered services.

As an example, suppose legacy printers are discovered that do not support the current AirPrint feature set. The proxy can alter the TXT record associated with the printer to add the necessary keys as well as any additional service records to allow AirPrint clients to discover and use the legacy printer.

As another example, suppose there is a printer that is behind a locked door where students do not have access. In this case, the printer's resource records MAY be filtered by the proxy so it does not show up during a browse operation on the subnet.

An Update proxy could have rulesets that define the translations it performs on the fly as is learns about matching services.

# DNS update

While DNS Update is well supported in authoritative DNS servers, it typically requires some form of authentication for the server to accept the update. The most common form is TSIG {{!RFC2845}},{{!RFC4635}} which is based on a shared secret and a one way hash over the contents of the record.

The Update proxy doesn't dictate a method of privacy or authentication for communication to an authoritative DNS Update server. However, implementations SHOULD ensure some form of authentication exists and even refuse to operate in an environment without authentication.

## Selection of authoritative unicast DNS server

The Update proxy should attempt to locate the authoritative DNS Update server for each subdomain in the following manner:

1. An Update proxy should first send an SRV query for _dns-update._udp.&lt;subdomain&gt;.&lt;domain&gt;. If an answer is received, the target and port number will provide the parameters needed for where to send updates.

    Note: _dns-update._tcp and _dns-update._tls-tcp have not yet been registered with IANA. However, this should not stop an Update proxy from attempting to connect to an authoritative DNS server via TLS/TCP or plain TCP. In fact, an SRV query for the TLS variant is encouraged and if no answers are returned but answers are returned for the _udp version, attempting to connect to the same target and the reserved port (853) for DNS over TLS as defined in Section 3.1 of {{!RFC7858}} is encouraged for privacy reasons.

2. The Update proxy can make a similar query for the same service in the domain if a subdomain specific answer isn't returned: _dns-update._udp.&lt;domain&gt;.

3. If no SRV records are returned, the Update proxy SHOULD consult local configuration policy to see if an DNS Update server has been configured.

4. If no local configuration exists for a DNS Update server, the Update proxy can query the NS records for the subdomain and try sending updates to the name server configured for the subdomain or for the domain. Again, using TLS/TCP is encouraged if available.

5. If DNS Updates are not accepted by the server(s) represented by the NS records, the the Update proxy can assume that DNS Updates are not available for the subdomain and it has no reason to listen for mDNS announcements on the IP subnet.

## DNS update sections

A DNS Update message contains four sections as specified in {{!RFC2136}}.

### Zone section

When an Update proxy is adding or removing services to/from a subdomain, the zone section MUST contain a single zone (ZOCOUNT = 1) and the ZNAME MUST be the subdomain being updated. ZTYPE MUST be SOA and ZCLASS MUST be the same class as the records being added/removed.

Updates to multiple subdomains MUST be performed in separate DNS Update messages with one subdomain per message.

If a new subdomain is being created for a domain by the Update proxy, the subdomain's parent zone should be used for the ZNAME. ZTYPE MUST be SOA and ZCLASS MUST be the same class as the subdomain's NS record CLASS that is going to be added. Similarly for removing a subdomain.

### Prerequisite section {#prereq}

It is not necessary for the Update proxy to include any prerequisites when adding/removing records. However, if the Update proxy wants to have better error handling, it can add prerequisites to ensure the state of the authoritative server is consistent.

Given that multiple Update proxies may exist for the same IP subnet (and subdomain), it is possible that similar records may be added or deleted to/from the authoritative server before the Update proxy's own messages are processed. This is not to be considered a fatal error and may happen during normal operation of redundant proxies. The use of prerequisite can be used to identify these cases if desired.

### Update section

The Update section contains all of the records that the proxy wants to be added/removed in a single subdomain. If TIMEOUT resource records are being manually added to the authoritative server, they MUST be included as regular resource records in the Update section. See {{lifetimes}} below for more information.

### Additional data section

The Update proxy may include additional data as needed. Instances where additional data might be included are:

1. When creating a subdomain by adding new NS records to a domain, `A` or `AAAA` glue records MAY be needed. Though, in most cases, the same authoritative server name / IP addresses should be used as in the parent domain.

2. If including a lease lifetime as discussed below in {{lifetimes}}, the OPT recording containing the Update lease will be sent in the additional data section.

3. The TSIG cryptographic signature of the DNS Update message should be the last resource record in the additional data section.

# DNS authoritative server behavior

The Update proxy will rely on the authoritative server to update the SERIAL number for the zone after each update is completed.

## DNS Push Notifications

An authoritative unicast DNS server MAY support DNS Push notifications {{?I-D.ietf-dnssd-push}} for client queries in order to provide more timely and more efficient responses. While this is outside of the scope of the Update proxy, it is mentioned here for completeness.

## DNSSEC compatibility

With mDNS, the next domain name field in an NSEC record could not reference the next record in the zone because it was not possible to know all of the records in the zone a priori. By mapping all known records into a unicast subdomain, the NSEC next domain name field can contain the next known record as defined. As new services are discovered and Updated in the authoritative unicast DNS server, the NSEC records can be kept up to date by the authoritative server.

The Update proxy will assume that DNS updates sent to zones with DNSSEC enabled will be updated as needed as specified in {{!RFC3007}}.

## DNS Update record lifetimes {#lifetimes}

When the Update proxy sends an DNS Update message to an authoritative unicast DNS server, it MAY include a lease lifetime to indicate how long the Update server should keep the resource records active in the zone. This is different from the TTL which tells resolvers how long to keep the records in their cache. Lease lifetimes may be based on different origin data. For example, when an IP address is assigned to a host via DHCP, the DHCP server will provide a time period for which the address is assigned to the host.

There are several possibilities for how a DNS Update server may limit the lifetime of records added via an update message.

1. The DNS update server MAY be configured to automatically delete the records after a certain fixed time period (such as 24 hours). This is a failsafe mechanism in case the origin of the record data goes offline and does not ever try to remove the records.

2. A lease lifetime can be communicated via an OPT record as defined in Dynamic DNS Update Leases {{?I-D.sekar-dns-ul}}. This provides a timeout period for all of the records added in the update message and is controlled by the sender of the update. This is a work in progress and does not yet have widespread adoption among authoritative unicast DNS server software.

3. Individual DNS TIMEOUT resource records {{?I-D.pusateri-dnsop-update-timeout}} can be added to the update message to indicate the timeout value for one or any number of the resource records contained in the update message. This is the most flexible but also does not have any adoption among authoritative unicast DNS server software. One advantage of the TIMEOUT resource records is that they are stored in the authoritative server like any other record and synchronized to secondary servers as well. Therefore, if the primary server were to restart or experience an extended failure, the lease lifetime would not be lost.

Note that it is possible to use both the Dynamic DNS Update leases to communicate the lease lifetime and for the authoritative unicast DNS server to create TIMEOUT resource records on demand to achieve the same result if the Update proxy does not include TIMEOUT resource records natively.

# Security Considerations

When a secure DNS Update is sent to an authoritative server, it should not be construed that this information is any more reliable than the original mDNS announcement was for which it was based. Care should always be taken when receiving mDNS announcements to ensure they are source IP address is one that belongs to an IP subnet on the received interface of the Update proxy. In addition, the TTL of the received link local announcement MUST be 1 to ensure it was not forwarded from a remote network.

Each Update proxy requires configuration of a shared secret for creation of the TSIG signature resource record contained as the last record in the Update message.

----

--- back

# Comparison to Discovery Proxy

The Update Proxy defined in this document is an alternative to the Discovery Proxy {{?I-D.ietf-dnssd-hybrid}} and the Discovery Relay {{?I-D.ietf-dnssd-mdns-relay}}. This solution makes different trade-offs than the ones made by the Discovery Proxy which offer some advantages at a cost of increased state.

The main difference is that the Discovery Proxy builds the list of matching services on demand by querying over mDNS and collecting the announcements in response to client queries. Whereas the Update proxy tries to build a complete list of services by listening for all announcements, discovering and refreshing them, and then inserting them into subdomains using DNS Update.

The main advantages of the Update proxy include limiting further propagation of IP multicast across the campus, providing a pathway to eliminate multicast entirely, faster response time to client queries, and the ability to provide DNSSEC signed security responses for client queries.

Another key difference is that the Update proxy never becomes an authoritative unicast DNS server for the attached subdomain. It simply updates the existing authoritative server for the domain. Therefore, the administrator is free to use existing authoritative DNS server infrastructure.

