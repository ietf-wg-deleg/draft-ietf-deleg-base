---
title: Extensible Delegation for DNS
abbrev: DELEG
category: std

docname: draft-ietf-deleg-latest
submissiontype: IETF
updates: 1034, 1035, 6672, 6840
number:
date:
consensus: true
v: 3
area: Internet
workgroup: deleg
keyword:
 - Internet-Draft
 - DNS
 - delegation
venue:
  group: deleg
  type: Working Group
  mail: dd@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/dd/
  github: ietf-wg-deleg/draft-ietf-deleg-base/
  latest: https://github.com/ietf-wg-deleg/draft-ietf-deleg-base/tree/gh-pages

ipr: trust200902

author:
-
    ins: T. April
    name: Tim April
    organization: Google, LLC
    email: ietf@tapril.net
-
    ins: P.Špaček
    name: Petr Špaček
    organization: ISC
    email: pspacek@isc.org
-
    ins: R.Weber
    name: Ralf Weber
    organization: Akamai Technologies
    email: rweber@akamai.com
-
    ins: D.Lawrence
    name: David C Lawrence
    organization: Salesforce
    email: tale@dd.org

--- abstract
This document proposes a new extensible method for delegation of the authority for a domain in Domain Name System (DNS) based in DELEG and DELEGI records.

A delegation in the DNS is a mechanism that enables efficient and distributed management of the DNS namespace.
The traditional DNS delegation is based on NS records which contain only hostname of a server and no other parameters.
New delegation records are extensible, can be secured with DNSSEC, and eliminate parent-child duplication from the DNS.

--- middle

# Introduction

In the Domain Name System, responsibility for each subdomain within the domain name hierarchy can be delegated to different servers, which makes them authoritative for their portion of the namespace.

The original DNS record that does this, called NS record, contains only hostname of a single nameserver and no other parameters.
The resolver needs to resolve these names into usable addresses and infer other required parameters, such as transport protocol and any other protocol features.
Moreover the NS record exists in two copies, at the delegation point, and in the child zone.
DNS Security Extension (DNSSEC) protect only one copy, in the child zone.

These properties of NS records limit resolvers to unencrypted UDP and TCP port 53, and this initial contact cannot be protected with DNSSEC.
Even if these two problems were somehow solved, NS record does not offer extensibility for any other parameters.
This limitation is a barrier for efficient introduction of new DNS technology.

The proposed DELEG and DELEGI record types remedy this problem by providing extensible parameters to indicate server capabilities and additional information, such as addresses that a resolver may use to reach the server.
The DELEG record creates a new delegation.
It is authoritative in the parent side of delegation and thus signed.
This makes it possible to validate all delegation parameters with DNSSEC, including all future extensions.
The DELEGI record is an auxiliary record which does not create delegation by itself, but can be used to share the same delegation information across any number of zones.
DELEGI is treated like regular authoritative records in their zone.

DELEG record can be used instead of or along side a NS record to create a delegation.
Combination of DELEG+NS is fully compatible with old resolvers.

Future documents can use the extensible mechanism for more advanced features like connecting to a name server with an encrypted transport.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in \\
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

Terminology regarding the Domain Name System comes from {{?BCP219}}, with addition terms defined here:

* legacy name servers: An authoritative server that does not support the DELEG record
* legacy resolvers: A resolver that does not support the DELEG record
* DELEG-aware: An authoritative server or resolver that follows the protocol defined in this document

# DELEG and DELEGI Record Types

The DELEG record (whose RRtype is TBD) Rdata contains a list of key-value pairs called "delegation information".
The delegation information has wire and display formats that are based on the rules in Appendix A of {{?RFC9460}}.
A DELEG record is authoritative for the zone in its owner name, and creates a delegation and thus lives in the parent of that zone.

The DELEGI record uses the same format as the DELEG record, but rather than defining a delegation,
a DELEGI is used as the target of the "include" mechanism.
Also unlike DELEG records, DELEGI records can live in any part of the DNS namespace.

When a resolver encounters an "include-name" key-value pair in a DELEG record, it queries for a DELEGI record.
Because a DELEGI record can itself contain an "include-name" key-value pair,
a resolver must be prepared to follow the "include-name" chain by making additional DELEGI queries.

Some delegation information key-value pairs are information about nameservers that a DELEG-aware resolver uses when it gets a DELEG or DELEGI record.
The keys defined in this document are described briefly here, and more fully described in {{nameserver-info}}.

* server-ip4: a set of IPv4 addresses for nameservers
* server-ip6: a set of IPv6 addresses for nameservers
* server-name: the domain name of a nameserver
* include-name: the domain name of a zone that has more information about the nameservers

TODO: Add some introduction comparing how resolvers see legacy delegation (set of NS and A/AAAA records) and DELEG delegation (DELEG and DELEGI records with server-ip4 and server-ip6 keys)

# Use of DELEG Records

A DELEG RRset MAY be present at a delegation point.
The DELEG RRset MAY contain multiple records.
DELEG RRsets MUST NOT appear at a zone's apex.
<!-- From the mailing list: Can we give advice on what an authoritative server should do if it encounters this
situation? Should it fail to load the zone? I can see that this is trying to say that
DELEGs do not follow the same model as legacy delegations with the NS RRSet in both
locations.
If so, perhaps it should just say that.
Regardless, I think a "MUST NOT" should
explain the consequences of not complying. -->

A DELEG RRset MAY be present with or without NS or DS RRsets at the delegation point.

## Resolvers

### Signaling DELEG Support

There will be both DELEG and NS needed for delegation for a long time.
Both legacy delegation (using NS records) and the DELEG protocol enable recursive resolution.
This document defines a new EDNS flag to signal that a resolver is DELEG-aware and therefore does not need NS records or glue information in a referral response.

A resolver that is DELEG-aware MUST signal that it supports the DELEG protocol by sending the DE bit when sending queries.
This bit is referred to as the "DELEG" (DE) bit.
In the context of the EDNS0 OPT meta-RR, the DE bit is the TBD of the "extended RCODE and flags" portion of the EDNS0 OPT meta-RR, structured as follows (to be updated when assigned):

                +0 (MSB)                +1 (LSB)
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
      0: |   EXTENDED-RCODE      |       VERSION         |
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
      2: |DO|CO|DE|              Z                       |
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+

Setting the DE bit to one in a query indicates the resolver understands new DELEG semantics and does not need NS records to follow a referral.
The DE bit set to 0 indicates the resolver is not DELEG-aware, and therefore can only be served referrals with NS records and other data according to pre-DELEG specifications.

### Referral

The DELEG record creates a zone cut similar to the NS record.

If one or more DELEG records exist at a given delegation point, a DELEG-aware resolver MUST treat the name servers from those DELEG records as authoritative for the child zone.
In such case, a DELEG-aware resolver MUST NOT use NS records even if they happen to be present in cache, even if resolution using DELEG records have failed for any reason.
Such fallback from DELEG to NS would invalidate security guarantees of DELEG protocol.

If no DELEG record exists at a given delegation point, DELEG-aware resolvers MUST use NS records as specified by {{!RFC1034}}.
See {{dnssec-validators}} for more information about protection from downgrade attacks.

### Parent-side types, QTYPE=DELEG

Record types defined as authoritative on the parent side of zone cut (currently DS and DELEG types) retain the same special handling as described in Section 2.6  of {{!RFC4035}}.

DELEG-unaware resolvers can get different types of answers for QTYPE=DELEG queries based on the configuration of the server, such as whether it is DELEG-aware and whether it also is authoritative for subdomains.

### Algorithm for "Finding the Best Servers to Ask" {#finding-best}

This document updates instructions for finding the best servers to ask.
That information currently is covered in Section 5.3.3 of {{!RFC1034}} and Section 3.4.1 of {{!RFC6672}} with the text "2. Find the best servers to ask."
Section 3.1.4.1 of {{!RFC4035}} should have explicitly updated Section 5.3.3 of {{!RFC1034}} for the DS RRtype, but failed to do so.
This document simply extends this existing behavior to DELEG RRtype as well, and makes this special case explicit.

When a DELEG RRset exists in a zone, DELEG-aware resolvers ignore the NS RRset for that zone.
This means that the DELEG-aware resolver ignores the NS RRset in the zone's parent as well as any cached NS RRset that the resolver might have gotten by looking in the apex of the zone.

Each delegation level can have a mixture of DELEG and NS RRtypes, and DELEG-aware resolvers MUST be able to follow chains of delegations which combines both types in arbitrary ways.

An example of a valid delegation tree:

    ; root zone with NS-only delegations
    . SOA ...
    test. NS ...

    ; test. zone with NS+DELEG delegations
    test. SOA ...
    sld.test. NS ...
    sld.test. DELEG ...

    ; sld.test. zone with NS-only delegation
    sld.test. SOA ...
    nssub.sld.test. NS ...

    ; nssub.sld.test. zone with DELEG-only delegation
    delegsub.sub.sld.test. DELEG ...

TODO: after the text below, refer back to this figure and show the order that a DELEG-aware resolver would take when there is a failure to find any good DELEG addresses at sub.sld.test, then any usable nameservers at sub.sld.test, and then maybe a good DELEG record at test.

The terms SNAME and SLIST used here are defined in Section 5.3.2 of {{!RFC1034}}:

SNAME is the domain name we are searching for.

SLIST is a structure which describes the name servers and the zone which the resolver is currently trying to query.
Neither {{RFC1034}} nor this document define how a resolver uses SLIST; they only define how to populate it.

A DELEG-aware SLIST needs to be able to hold two types of information: delegations defined by NS records and delegations defined by DELEG records.
DELEG and NS delegations can create cyclic dependencies and/or lead to duplicate entries which point to the same server.
Resolvers need to enforce suitable limits to prevent damage even if someone has incorrectly configured some of the data used to create an SLIST.

This leads to a modifications of the description from earlier documents for DELEG-aware resolvers can find the best servers to ask.
That description becomes:

1. Determine deepest possible zone cut which can potentially hold the answer for given (query name, type, class) combination:

    1. Start with SNAME equal to QNAME.

    1. If QTYPE is a type that is authoritative at the parent side of a zone cut (currently, DS or DELEG), remove the leftmost label from SNAME.
For example, if the QNAME is "test.example." and the QTYPE is DELEG or DS, set SNAME to "example.".

1. Look for locally-available DELEG and NS RRsets, starting at current SNAME.

    1. For given SNAME, check for existence of a DELEG RRset.
If it exists, the resolver MUST use its content to populate SLIST.
However, if the DELEG RRset is known to exist but is unusable (for example, if it is found in DNSSEC BAD cache), the resolver MUST NOT instead use an NS RRset, even if it is locally available; instead, the resolver MUST treat this case as if no servers were available.

    1. If a given SNAME is proven to not have a DELEG RRset but does have NS RRset, the resolver MUST copy the NS RRset into SLIST.

    1. If SLIST is now populated, stop walking up the DNS tree.
However, if SLIST is not populated, remove leftmost label from SNAME and go back to the first step, using the new (shortened) SNAME.
Do not go back to the first step if doing so would exceed the amount of work that the resolver is configured to do when processing names; see {{too-much-work}}.

The rest of the Step 2's description is not affected by this document.

TODO: Determine what to do about ". DELEG" or ". DS" queries, which by definition do not have answers in the zone.

### Nameserver Information for Delegation {#nameserver-info}

The DELEG and DELEGI records have four keys that describe information about nameservers.
The purpose of this information is to populate the SLIST with IP addresses of the nameservers for a zone.
The types of information defined in this document are:

* server-ip4: a set of IPv4 addresses for nameservers
* server-ip6: a set of IPv6 addresses for nameservers
* server-name: the domain name of a nameserver; the addresses must be fetched
* include-name: the domain name of a zone that has more information about the nameservers

The presentation values for server-ip4 and server-ip6 are comma-separated list of one or more IP addresses of the appropriate family in standard textual format {{?RFC5952}} {{?RFC4001}}.
The wire formats for server-ip4 and server-ip6 are a sequence of IP addresses in network byte order (for the respective address family).

The presentation values for server-name and include-name are as full-qualified domain names.
The wire formats are the same as the wire formats for domain names, and MUST NOT be compressed.

TODO: Are they? Are we going to forbid normal zone file expansion where names without trailing . get current origin appended to them?

If any one of these keys is used, it MUST have a value (that is, it cannot be a key with a zero-length value).

A DELEG or DELEGI record SHOULD carry exactly one set of server information, chosen from the following:

- one server-ip4 key
- one server-ip6 key
- a pair consisting of one server-ip4 and one server-ip6
- one server-name key
- one include-name key

When using server-name, the addresses must be fetched using normal DNS resolution.
This means the value of the server-name key MUST NOT be inside the delegated domain.

### Populating the SLIST from DELEG and DELEGI Records

Each individual DELEG record inside a DELEG RRset, or each individual DELEGI record in a DELEGI RRset, can cause the addition of zero or more entries to SLIST.

A resolver processes each individual DELEG record within a DELEG RRset, or each individual DELEGI record in a DELEGI RRset, using the following steps:

1. If one or more server-ip4 or server-ip6 keys are present inside the record, copy all the address values from either key into SLIST.
Ignore any server-name or include-name keys that are (erroneously) present in the same record.
Stop processing this record.

1. If a server-name key is present in the record, resolve it into addresses.
Copy these addresses into SLIST.
Ignore any include-name keys that are (erroneously) present in the same record.
Stop processing this record.

1. If a include-name key is present in the record, resolve it into a DELEGI RRset.
Recursively apply the algorithm described in this section, after checking that the maximum loop count described in {{too-much-work}} has not been reached.

1. If none of the above applies, SLIST is not modified by this particular record.

A DELEG-aware resolver MAY implement lazy filling of SLIST, such as by deferring processing remaining records if SLIST already has what the resolver considers a sufficiently large pool of addresses to contact.

## Authoritative Servers

DELEG-aware authoritative servers act differently when handling queries from DELEG-unaware clients (those with DE=0) than from DELEG-aware clients (those with DE=1).

The server MUST copy the value of the DE bit from the query into the response.

### DELEG-unaware Clients

DELEG-unaware clients do not use DELEG records for delegation.
When a DELEG-aware authoritative server responds to a DELEG-unaware client, any DELEG record in the response does not create a zone cut, is not returned in referral responses, and is not considered authoritative on the parent side of a zone cut.
Because of this, DELEG-aware authoritative servers MUST answer as if they are DELEG-unaware.
Please note this instruction does not affect DNSSEC signing, i.e. no special handling for NSEC type bitmap is necessary and DELEG RRtype is accurately represented even for DELEG-unaware clients.

Two specific cases of DELEG-aware authoritative responding in DELEG-unaware manner are described here.

#### DELEG-unaware Clients Requesting QTYPE=DELEG

In DELEG-unaware clients, records with the DELEG RRtype are not authoritative on the parent side.
Thus, queries with DE=0 and QTYPE=DELEG MUST result in a normal legacy response, such as a legacy delegation if there are NS records.

TODO: Should we have an example with auth having parent+child zone at the same time, and DE=0 QTYPE=DELEG query?

#### DELEG-unaware Clients with DELEG RRs Present but No NS RRs

DELEG-unaware clients might ask for a name which belongs to a zone delegated only with DELEG RRs (that is, without any NS RRs).
Such zone is, by definition, not resolvable for DELEG-unaware clients.
In this case, the DELEG record itself cannot create a zone cut, and the DELEG-aware authoritative server MUST return a legacy response.

<!-- From the mailing list: What is a "legacy response" when there are no NS RRs present? NXDOMAIN? Can this be spelled out? -->

The legacy response might be confusing for subdomains of zones which actually exist because DELEG-aware clients would get a different answer, namely a delegation.
An example of a legacy response is in {{legacynxdomain}}.

The authoritative server is RECOMMENDED to supplement these responses to DELEG-unaware resolvers with Extended DNS Error "New Delegation Only".

TODO: debate if WG wants to do explicit SERVFAIL for this case instead of 'just' EDE.

### DELEG-aware Clients

When the client indicates that it is DELEG-aware by setting DE=1 in the query, DELEG-aware authoritative servers treat DELEG records as zone cuts, and the servers are authoritative on parent side of zone cut.
This new zone cut has priority over legacy delegation with NS RRset.

#### DELEG-aware Clients Requesting QTYPE=DELEG

An explicit query for the DELEG RRtype at a delegation point behaves much like query for the DS RRtype: the server answers authoritatively from the parent zone.
All previous specifications for special handling queries with QTYPE=DS apply equally to QTYPE=DELEG.
In summary, server either provides an authoritative DELEG RRset or proves its non-existence.

#### Delegation with DELEG

If the delegation has a DELEG RRset, the authoritative server MUST put the DELEG RRset into the Authority section of the referral.
In this case, the server MUST NOT include the NS RRset into the Authority section.
Presence of the covering RRSIG follows the normal DNSSEC specification for answers with authoritative zone data.

Similarly, rules for DS RRset inclusion into referrals apply as specified by DNSSEC protocol.

#### DELEG-aware Clients with NS RRs Present but No DELEG RRs

If the delegation does not have a DELEG RRset, the authoritative server MUST put the NS RRset into the authority section of the referral.
The absence of the DELEG RRset MUST be proven as specified by DNSSEC protocol for authoritative data.

Similarly, rules for DS RRset inclusion into referrals apply as specified by the DNSSEC protocol.
Please note in practice the same process and records are used to prove non-existence of DELEG and DS RRsets.

## DNSSEC Signers

The DELEG record is authoritative on the parent side of a zone cut and needs to be signed as such.
Existing rules from DNSSEC specification apply.
In summary: for DNSSEC signing, treat the DELEG RRtype the same way as the DS RRtype.

In order to protect validators from downgrade attacks this draft introduces a new DNSKEY flag ADT (Authoritative Delegation Types).
In zones which contain a DELEG RRset, this flag MUST be set to 1 in at least one of the DNSKEY records published in the zone.

## DNSSEC Validators {#dnssec-validators}

DELEG awareness introduces additional requirements on validators.

### Clarifications on Nonexistence Proofs

This document updates Section 4.1 of {{!RFC6840}} to include "NS or DELEG" types in type bitmap as indication of a delegation point, and generalizes applicability of ancestor delegation proof to all RRtypes that are authoritative at the parent (that is, both DS and DELEG).
The text in that section is updated as follows:

An "ancestor delegation" NSEC RR (or NSEC3 RR) is one with:

-  the NS and/or DELEG bit set,

-  the Start of Authority (SOA) bit clear, and

-  a signer field that is shorter than the owner name of the NSEC RR,
   or the original owner name for the NSEC3 RR.

Ancestor delegation NSEC or NSEC3 RRs MUST NOT be used to assume
nonexistence of any RRs below that zone cut, which include all RRs at
that (original) owner name other than types authoritative at the parent-side of
zone cut (DS and DELEG), and all RRs below that owner name regardless of
type.

### Insecure Delegation Proofs

This document updates Section 4.4 of {{!RFC6840}} to include securing DELEG records, and explicitly states that Opt-Out is not applicable to the DELEG protocol.
The first paragraph of that section is updated to read:

Section 5.2 of {{!RFC4035}} specifies that a validator, when proving a
delegation is not secure, needs to check for the absence of the DS
and SOA bits in the NSEC (or NSEC3) type bitmap.
The validator also
MUST check for the presence of the NS or the DELEG bit in the matching NSEC (or
NSEC3) RR (proving that there is, indeed, a delegation).
Alternately, the validator must make sure that the delegation with NS record is covered by an NSEC3
RR with the Opt-Out flag set.
Opt-Out is not applicable to DELEG RRtype
because DELEG records are authoritative at the parent side of a zone cut in the same
way that DS RRtypes are.

### Referral downgrade protection

If a zone is secured by DNSSEC, and if any DNSKEY record in the zone has the ADT flag set to 1, a DELEG-aware validator MUST prove the absence of a DELEG RRset in referral responses from this particular zone.

Without this check, an attacker could strip the DELEG RRset from a referral response and replace it with an unsigned (and potentially malicious) NS RRset.
A referral response with an unsigned NS and signed DS RRsets does not require additional proofs of nonexistence according to pre-DELEG DNSSEC specification, and it would have been accepted as a delegation without DELEG RRset.

### Chaining

A Validating Stub Resolver that is DELEG-aware has to use a Security-Aware Resolver that is DELEG-aware and, if it is behind a forwarder, that forwarder has to be security-aware and DELEG-aware as well.

# Security Considerations

TODO: Add more here

## Preventing Over-work Attacks {#too-much-work}

Resolvers MUST prevent situations where accidental misconfiguration of zones or malicious attacks cause them to perform too much work when resolving.
This document describes two sets of actions that, if not controlled, could lead to over-work attacks:

- Names with many subdomains can cause walking up the tree to populate SLIST ({{finding-best}}) to be burdensome.
To prevent this, the resolver SHOULD NOT walk up more than %%TODO: come up with a number%% labels in order to contribute to SLIST.

- Long chains of include-name information ({{nameserver-info}}), and those with circular chains if include-name information, can be burdensome.
To prevent this, the resolver SHOULD NOT follow more than 3 include-name chains in an RRset when populating SLIST.
Note that include-name chains can have CNAME steps in them; in such a case, a CNAME step is counted the same as a DELEGI step when determining when to stop following a chain.

# IANA Considerations

## Changes to Existing Registries

IANA is requested to allocate the DELEG RR and the DELEGI RR in the Resource Record (RR) TYPEs registry, with the meaning of "enhanced delegation information" and referencing this document.

IANA is requested to assign a new bit in the DNSKEY RR Flags registry ({{!RFC4034}}) for the ADT bit (N), with the description "Authoritative Delegation Types" and referencing this document.
For compatibility reasons we request the bit 14 to be used.
This value has been proven to work whereas bit 0 was proven to break in practical deployments (because of bugs).

IANA is requested to assign a bit from the EDNS Header Flags registry ({{!RFC6891}}), with the abbreviation DE, the description "DELEG enabled" and referencing this document.

IANA is requested to assign a value from the Extended DNS Error Codes ({{!RFC8914}}), with the Purpose "New Delegation Only" and referencing this document.

## New Registry for Delegation Information

IANA is requested to create the "DELEG Delegation Information" registry.
This registry defines the namespace for delegation information keys, including string representations and numeric key values.

### Procedure

A registration MUST include the following fields:

Number:  Wire-format numeric identifier (range 0-65535)
Name:  Unique presentation name
Meaning:  A short description
Reference:  Location of specification or registration source
Change Controller:  Person or entity, with contact information if appropriate

The characters in the registered Name field entry MUST be lowercase alphanumeric or "-".
The name MUST NOT start with "key".

The registration policy for new entries is Expert Review ({{!RFC8126}}).
The designated expert MUST ensure that the reference is stable and publicly available and that it specifies how to convert the delegation information's presentation format to wire format.
The reference MAY be any individual's Internet-Draft or a document from any other source with similar assurances of stability and availability.
An entry MAY specify a reference of the form "Same as (other key name)" if it uses the same presentation and wire formats as an existing key.

This arrangement supports the development of new parameters while ensuring that zone files can be made interoperable.

### Initial Contents

The "DELEG Delegation Information" registry should be populated with the following initial registrations:

~~~
Number:  1
Name:  server-ip4
Meaning:  A set of IPv4 addresses of nameservers
Reference:  {{nameserver-info}} of this document
Change Controller:  IETF

Number:  2
Name:  server-ip6
Meaning:  A set of IPv6 addresses of nameservers
Reference:  {{nameserver-info}} of this document
Change Controller:  IETF

Number:  3
Name:  server-name
Meaning:  The fully-qualified domain name of a nameserver
Reference:  {{nameserver-info}} of this document
Change Controller:  IETF

Number:  4
Name:  include-name
Meaning:  The fully-qualified domain of a DELEGI record
Reference:  {{nameserver-info}} of this document
Change Controller:  IETF

The registration for number 0 is reserved.
The registration for numbers 65280-65535 is reserved for private use.
~~~

--- back

#  Examples

The following example shows an excerpt from a signed root zone.
It shows the delegation point for "example." and "test."

The "example." delegation has DELEG and NS records.
The "test." delegation has DELEG but no NS records.

TODO: Examples of using server-ip4 and server-ip6.
Also, examples that show DELEGI records in ns2.example.net and ns3.example.org.

    example.   300 IN DELEG server-name=a.example.
    example.   300 IN DELEG include-name=ns2.example.net.
    example.   300 IN DELEG include-name=ns3.example.org.
    example.   300 IN RRSIG DELEG 13 4 300 20250214164848 (
                            20250207134348 21261 . HyDHYVT5KcqWc7J..= )
    example.   300 IN NS    a.example.
    example.   300 IN NS    b.example.net.
    example.   300 IN NS    c.example.org.
    example.   300 IN DS    65163 13 2 5F86F2F3AE2B02...
    example.   300 IN RRSIG DS 13 4 300 20250214164848 (
                            20250207134348 21261 . O0k558jHhyrC21J..= )
    example.   300 IN NSEC  a.example. NS DS RRSIG NSEC DELEG
    example.   300 IN RRSIG NSEC 13 4 300 20250214164848 (
                            20250207134348 21261 . 1Kl8vab96gG21Aa..= )
    a.example. 300 IN A     192.0.2.1
    a.example. 300 IN AAAA  2001:DB8::1

The "test." delegation point has a DELEG record and no NS record.

    test.      300 IN DELEG include-name=ns2.example.net
    test.      300 IN RRSIG DELEG 13 4 300 20250214164848 (
                            20250207134348 21261 . 98Aac9f7A1Ac26Q..= )
    test.      300 IN NSEC  a.test. RRSIG NSEC DELEG
    test.      300 IN RRSIG NSEC 13 4 300  20250214164848 (
                            20250207134348 21261 . kj7YY5tr9h7UqlK..= )

## Responses

The following sections show referral examples:

## DO bit clear, DE bit clear

### Query for foo.example

    ;; Header: QR RCODE=0
    ;;

    ;; Question
    foo.example.  IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    example.   300 IN NS    a.example.
    example.   300 IN NS    b.example.net.
    example.   300 IN NS    c.example.org.

    ;; Additional
    a.example. 300 IN A     192.0.2.1
    a.example. 300 IN AAAA  2001:DB8::1

### Query for foo.test

    ;; Header: QR AA RCODE=3
    ;;

    ;; Question
    foo.test.   IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    .   300 IN SOA ...

    ;; Additional
    ;; OPT with Extended DNS Error: New Delegation Only


## DO bit set, DE bit clear

### Query for foo.example


    ;; Header: QR DO RCODE=0
    ;;

    ;; Question
    foo.example.   IN MX

    ;; Answer
    ;; (empty)

    ;; Authority

    example.   300 IN NS    a.example.
    example.   300 IN NS    b.example.net.
    example.   300 IN NS    c.example.org.
    example.   300 IN DS    65163 13 2 5F86F2F3AE2B02...
    example.   300 IN RRSIG DS 13 4 300 20250214164848 (
                            20250207134348 21261 . O0k558jHhyrC21J..= )
    ;; Additional
    a.example. 300 IN A     192.0.2.1
    a.example. 300 IN AAAA  2001:DB8::1


### Query for foo.test {#legacynxdomain}

    ;; Header: QR DO AA RCODE=3
    ;;

    ;; Question
    foo.test.      IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    .          300 IN SOA ...
    .          300 IN RRSIG SOA ...
    .          300 IN NSEC  aaa NS SOA RRSIG NSEC DNSKEY ZONEMD
    .          300 IN RRSIG NSEC 13 4 300
    test.      300 IN NSEC  a.test. RRSIG NSEC DELEG
    test.      300 IN RRSIG NSEC 13 4 300  20250214164848 (
                            20250207134348 21261 . aBFYask;djf7UqlK..= )

    ;; Additional
    ;; OPT with Extended DNS Error: New Delegation Only


## DO bit clear, DE bit set

### Query for foo.example


    ;; Header: QR DE RCODE=0
    ;;

    ;; Question
    foo.example.  IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    example.   300 IN DELEG server-name=a.example.
    example.   300 IN DELEG include-name=ns2.example.net.
    example.   300 IN DELEG include-name=ns3.example.org.

    ;; Additional
    ;; (empty)

### Query for foo.test

    ;; Header: QR AA RCODE=0
    ;;

    ;; Question
    foo.test.   IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    test.      300 IN DELEG include-name=ns2.example.net

    ;; Additional
    ;; (empty)

## DO bit set, DE bit set

### Query for foo.example

    ;; Header: QR DO DE RCODE=0
    ;;

    ;; Question
    foo.example.  IN MX

    ;; Answer
    ;; (empty)

    ;; Authority

    example.   300 IN DELEG server-name=a.example.
    example.   300 IN DELEG include-name=ns2.example.net.
    example.   300 IN DELEG include-name=ns3.example.org.
    example.   300 IN RRSIG DELEG 13 4 300 20250214164848 (
                            20250207134348 21261 . HyDHYVT5KcqWc7J..= )
    example.   300 IN DS    65163 13 2 5F86F2F3AE2B02...
    example.   300 IN RRSIG DS 13 4 300 20250214164848 (
                            20250207134348 21261 . O0k558jHhyrC21J..= )

    ;; Additional
    a.example. 300 IN A     192.0.2.1
    a.example. 300 IN AAAA  2001:DB8::1

### Query for foo.test

    ;; Header: QR DO DE AA RCODE=0
    ;;

    ;; Question
    foo.test.      IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    test.      300 IN DELEG include-name=ns2.example.net.
    test.      300 IN RRSIG DELEG 13 4 300 20250214164848 (
                            20250207134348 21261 . 98Aac9f7A1Ac26Q..= )
    test.      300 IN NSEC  a.test. RRSIG NSEC DELEG
    test.      300 IN RRSIG NSEC 13 4 300  20250214164848 (
                            20250207134348 21261 . kj7YY5tr9h7UqlK..= )

    ;; Additional
    ;; (empty)


# Acknowledgments
{:numbered="false"}

This document is heavily based on past work done by Tim April in
{{?I-D.tapril-ns2}} and thus extends the thanks to the people helping on this which are:
John Levine, Erik Nygren, Jon Reed, Ben Kaduk, Mashooq Muhaimen, Jason Moreau, Jerrod Wiesman, Billy Tiemann, Gordon Marx and Brian Wellington.

Work on DELEG protocol has started at IETF 118 Hackaton.
Hackaton participants: Christian Elmerot, David Blacka, David Lawrence, Edward Lewis, Erik Nygren, George Michaelson, Jan Včelák, Klaus Darilion, Libor Peltan, Manu Bretelle, Peter van Dijk, Petr Špaček, Philip Homburg, Ralf Weber, Roy Arends, Shane Kerr, Shumon Huque, Vandan Adhvaryu, Vladimír Čunát, Andreas Schulze.

Other people joined the effort after the initial hackaton: Ben Schwartz, Bob Halley, Paul Hoffman, ...


# TODO

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

* Write a security considerations section
* Change the parameters from temporary to permanent once IANA assigned. Temporary use:
  * DELEG QType code is 65432
  * DELEG EDNS Flag Bit is 3
  * DELEG DNSKEY Flag Bit is 14
