% title = "Privacy respecting Traffic Steering in DNS answers"
% abbrev = "AS-traffic 
% category = "std"
% ipr="trust200902"
% docName = "draft-ogud-privacy-DNS-traffic-steering-00.txt"
% workgroup = "dnsop"
% area = "Operations" 
% keyword = ["dns", "traffic sterring", "privacy"]
%
% date = 2018-04-16T00:00:00Z
%
% [[author]]
% fullname = "Marek Vavrusa"
% initials = "M."
% surname = "Vavrusa"
% organization="Cloudflare"
%   [author.address]
%   email="mvavrusa@cloudflare.com"
%   [author.address.postal]
%   city="San Francisco"
%   region="CA"
%
% [[author]]
% initials = "O."
% surname = "Gudmundsson"
% fullname = "Olafur Gudmundsson"
% organization = "Cloudflare, Inc."
%  [author.address]
%  email = "olafur+ietf@cloudflare.com"
%  [author.address.postal]
%  city = "San Francisco"
%  region = "CA"
%
% 

.# Abstract 
ECS is bad 

{mainmatter}

# Introduction 

This document is not about if DNS is the right place to steer traffic on the Interent. 
It must be said that DNS is a blunt instrument for traffic steering for various reasons.  Services and applications ought to be able to redirect+reconnect to closer servers, once the first connections have been established if that is beneficial. 

## Client Subnet 
There is an Informational RFC about providing information about the edge clieant asking Recursive Resolver to look up a RRset[@RFC7871]. This was a controversial RFC thus it was not on the standards track. Traffic Engineering community has embrached this EDNS0 option [@RFC6891], as well as by tacking entities, DNS community has been somewhere between hostile and do-not-care. 
The main arguments for this is "it works", get over it. 
The main argument against it is that the cost of the processing is pushed out to the resolvers in the form of 
  - Cache explosition, multiple RRset's for the Qname+Qtype+Qclass with a differenciator of network mask 
  - Short TTL's to deal with load on services 
  - Badly formatted option with strange fields, 
  - Leaks information on random queries, no limits on when option is set 

Intent vs Preloading of answers
There are numerous examples of tools like browsers that try to speed up service by asking for all the links on a page, even if the user has not clicked on any of them. This can leak information, just like all the trackers that are loaded on the page. 


## RFC2119 Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [@RFC2119].

# Proposal 
In gerneral when Unicast is used, there is a "desire" by traffic engineers to give out a server address that is close to the customer that is asking about the server, to reduce latency and make eye-balls happy[@RFC6555]. In the case of a customer using a "local ISP" provided resolver this is not an issue the address of the resolver is sufficient for the Authority to determine closeness and select the answer. 

When resolvers that cover large areas are used on the other hand things get harder for traffic sterring, this proposal is about creating a better framework that allows DNS and Trfaffic Engineering to coexist in a more privacy centric world. 

## New option AS-source 
IANA is requested to allocate a new EDNS0 option (value TBD), named "ASsorce"
This option contains 3 fields with fixed size of 12 bytes. 
  -- AS number the query originated in 4 bytes
  -- Resolver/Client Airport value  1 byte;  Resolver 1 Client 2 Hidden 0 
  -- IATA Airport code  7 byes 

The AS number is the AS# the client address is advertised from,  the IATA airport code is the code for the nearest Airport either to Resolver or preferably to the client address as indicated by the flag. 

If the authority wants this value to be reused only that AS then it echos back the ASsource option,
If the answer can be used for all AS's in that location it sets the AS value to 0. 
If the answer can be used for other AS's there can be multiple ASSource options in the answer. 


This option SHOULD never be set by stub-resolver. This option MUST only be sent on a query for anthing queries for A or AAAA records. 

Upstream MUST NOT send ASsource response on a query with out that option. 

## Discovering support by upstream

Before sending this option Resolver should check if upstream Resolver or Authority supports it by responding to the specified probe query 
    Qname: assource.server. QCLASS=CHAOS Qtype:A  OPT = ASSource { 0 1 "1234567"}
 with following response 
    QNAME: assource.server. QCLASS=CHAOS Qtype:A   Value=127.123.45.6.7  OPT= ASSource { 0 1 }

Any other response MUST be treated as Upstream does not support this option. 


## Protocol changes 
