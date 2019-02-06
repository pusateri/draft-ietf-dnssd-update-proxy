---
title: DNS Update Proxy for mDNS
docname: draft-ietf-dnssd-update-proxy-00
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

This document describes a method to dynamically map multicast DNS advertisements into the unicast DNS namespace for use by service discovery clients. It does not define any new protocols but uses existing DNS protocols in new ways. This solves existing problems with service discovery across multiple IP subnets in a simple, yet efficient, manner.

--- middle

# Introduction

Multicast DNS is used today for link-local service discovery. While this has worked reasonably well on the local link, current deployment reveals two problems. First, mDNS wasn't designed to traverse across multi-subnet campus networks. Second, IP multicast doesn't work across all link types and can be problematic on 802.11 Wifi networks. Therefore, a solution is desired to contain legacy multicast DNS service discovery and transition to a unicast DNS service discovery model. By mapping the current mDNS discovered services into regular unicast DNS authoritative servers, clients from any IP subnet can make unicast queries through normal unicast DNS resolvers.

There are many ways to map services discovered using multicast DNS into the unicast namespace. This document describes a way to do the mapping using a proxy that sends DNS Update messages directly to a unicast DNS authoritative server. While it is possible for each service advertiser to send it's own DNS Update, key management has prevented widespread deployment of DNS Updates across a domain. By having a limited number of proxies sitting on one or more IP subnets, it is possible to provide secure DNS updates at a manageable scale. Future work to automate secure DNS Updates on a larger scale is needed.

This document will explain how services on each .local domain will be mapped into the unicast DNS namespace and how unicast clients will discover these services. It is important to note that no changes are required in either the clients, DNS authoritative servers, or DNS resolver infrastructure. In addition, while the Update Proxy is a new logical concept, it requires no new protocols to be defined and can be built using existing DNS libraries.

# Requirements Language

{::boilerplate bcp14+}

# DNS subdomain model

Each .local domain which logically maps to an IP subnet is modeled as a separate subdomain in the unicast DNS hierarchy. Each of these subdomains must be browseable (respond to PTR queries for b._dns-sd._udp.&lt;subdomain&gt;.&lt;domain&gt;.). See Section 11 of {{!RFC6763}} for more details about browsing. These subdomains are typically special use subdomains for mDNS mappings.

## Subdomain naming {#subdomain}

The browseable subdomain label is prepended to the domain name and seperated by a period. See {{?RFC7719}} for more information on subdomains and labels. It is not important that the label be human readable or have organizational significance. End users will not be interacting with these labels. The main requirement is that they be unique within the domain for each IP subnet. Subdomain labels can be obtained by the proxy in several ways. The following methods should be attempted in order to assure consistency amoung redundant proxies:

1. PTR query for IP subnet through local resolver

    As an example, suppose a proxy was connected to IPv4 subnet 203.0.113.0/24. In order to determine if there was a subdomain name for this subnet, the proxy would issue a PTR query for 0.113.0.203.in-addr.arpa. If a response is returned with an answer, then the name in the answer should be the subdomain name including the domain name for the network.

2. proxy local configuration override

    If no answer is returned, the proxy may have local configuration containing a subdomain name for the network. If so, this subdomain should be used.

3. algorithmic subdomain label generation

    If no local configuration is present for the IP subnet, the proxy may generate a unique label and use that for the subdomain by appending a common domain name. One such algorithm is to take the network form of an IPv4 subnet without a prefix length (host portion all zeros) and convert it to hexidecimal. This will give a 8 character unique string to use as a subdomain label.

## Domain name discovery

The base domain name to use for each subdomain also has to be discovered on a per IP subnet basis. In most cases, the domain name will be the same for all IP subnets because they are all contained in a single administrative domain. However, this is not required and a proxy administrator may need to span multiple administrative boundaries requiring different domain names on different IP subnets (and therefore, subdomains).

There is not a direct query to discover a separate domain name but the domain name is included with the subdomain in the response to the PTR query above in {{subdomain}}. If the PTR query returns an empty response, then the domain name can be obtained from local proxy configuration and if no domain name is specified there, the default domain for the host should be used.

## Client service discovery

Fortunately, clients performing service discovery require no changes in order to work with the Update proxy. Existing clients already support wide-area bonjour which specifies how to query search domains and subdomains for services. See section 11 of {{!RFC6763}}.

# Update proxy behavior

Since no new protocols are defined, this document mostly describes the expected behavior of the Update proxy and how it uses existing protocols to achieve multi IP subnet service discovery. The behavior is mostly intuitive but is described to ensure compatibility and completeness.

## mDNS service announcements

The Update proxy should listen to mDNS service announcements (responses) on all interfaces it is proxying for. Multiple Update proxies can be active on the same IP subnet at the same time. See {{!RFC6762}} for more information on multicast DNS.

## Service caching and refresh

As specified in Section 8.3 of {{!RFC6762}} service announcements are sent multiple times for redundancy. However, there is no need to send duplicate Update messages to the authoritative unicast DNS server. Therefore, the Update proxy should cache service announcements and only send DNS Update messages when needed.

As described in Section 8.4 of {{!RFC6762}}, a host may send "goodbye" announcements by setting the TTL to 0. In this case, the record MUST be removed from the cache or otherwise marked as expired and a DNS Update should be sent to the authoritative unicast DNS server removing the record.

The Update proxy MUST also remove/expire old cache entries and remove the records from the unicast authoritative DNS server when the cache-flush bit is set on new announcements as described in Section 10.2 of {{!RFC6762}}.

<!-- Add text about refreshing entries before TTL expires -->

## mDNS probing

While {{!RFC6762}} recommends all potential answers be included in mDNS probe queries, because these records haven't gone through conflict resolution, they should not be regarded as announcements of services. Therefore, an Update proxy MUST NOT rely on information in any section of DNS query messages.

## Link-local addressing

In the IPv6 case, the source address of the announcements is a link-local IPv6 address that will probably be different than the IP subnet that the service is being provided on. However, it is certainly possible that link-local addressing is used with IPv4 as well. This is not as common but exists in a zero-conf environment where no IPv4 addresses are assigned via DHCP or statically and the hosts revert to link-local IPv4 addresses (169.254/16), see {{!RFC3927}}.

If the service SRV target resolves to only a link-local address, then the service is not eligble to be advertised outside of the link and shouldn't be sent to the authoritative unicast DNS server by the Update proxy.

In general, the Update proxy needs to ensure that the service is reachable outside of the link it is announced on before sending an Update to the authoritative server for the subdomain.

## IPv6 and IPv4 on same link

Announced services may be available on IPv4, IPv6, or both on the same link. If both IPv4 'A' records {{!RFC1035}} and IPv6 'AAAA' records {{!RFC3596}} are published for an SRV target {{!RFC2782}} name, the administrator should provide the service over both protocols.

In some cases, this won't be possible. This will not incur any extra delays if clients attempt connections over both IPv4 and IPv6 protocols simultaneously but if one protocol is preferred over another, delays may occur.

## multiple logical IP subnets

Multiple IP subnets on the same link are just a more general case of IPv4 and IPv6 on the same link. When multiple IP subnets exist for the same protocol on the same link, they appear as separate interfaces to the Update proxy and require a separate subdomain name just as IPv4 and IPv6 do.

This is required for a client on one logical IP subnet of an interface to communicate with a service provided by a host on a different IP subnet of the same link.

If a SRV target resolves to addresses on multiple logical IP subnets of the same interface, the service can be included in multiple subdomains on the appropriate server(s) for those subdomains.

## Proxy redundancy

T

## Service filtering and translation

# DNS update messages

## Selection of unicast authoritative DNS server

## DNS update prerequisites

## Cryptographically signed update messages

# DNS authoritative server behavior

## DNSSEC compatibility

## Lease lifetimes

## Timeout resource records

# Comparison to Discovery Proxy

The Update Proxy defined in this document is an alternative to the Discovery Proxy {{?I-D.ietf-dnssd-hybrid}} and the Discovery Relay {{?I-D.ietf-dnssd-mdns-relay}}. This solution makes different trade-offs than the ones made by the Discovery Proxy which offer substantial advantages at a cost of increased state.

These advantages include limiting further propagation of IP multicast across the campus, providing a pathway to eliminate multicast entirely, faster response time to client queries, and the ability to provide DNSSEC signed security responses for client queries.


--- back
