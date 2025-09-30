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
This document proposes a new extensible method for the delegation of authority for a domain in the Domain Name System (DNS) using DELEG and DELEGI records.

A delegation in the DNS enables efficient and distributed management of the DNS namespace.
The traditional DNS delegation is based on NS records which contain only hostnames of servers and no other parameters.
The new delegation records are extensible, can be secured with DNSSEC, and eliminate the problem of having two sources of truth for delegation information.

--- middle

# Introduction

In the Domain Name System, responsibility for each subdomain within the domain name hierarchy can be delegated to different servers, which makes them authoritative for their portion of the namespace.

The original DNS record that does this, called an NS record, contains only the hostname of a single name server and no other parameters.
The resolver needs to resolve these names into usable addresses and infer other required parameters, such as the transport protocol and any other protocol features.
Moreover, the NS record set exists in two places--one at the delegation point, and the other, possibly different, in the child zone.
The DNS Security Extensions (DNSSEC) protect only one copy, those in the child zone.

These properties of NS records limit resolvers to unencrypted UDP and TCP port 53, and this initial contact cannot be protected with DNSSEC.
Even if these two problems were somehow solved, the NS record does not offer extensibility for any other parameters.
This limitation is a barrier for the efficient introduction of new DNS technology.

The proposed DELEG and DELEGI resource record (RR) types remedy this problem by providing extensible parameters to indicate server capabilities and additional information, such as other transport protocols that a resolver may use.

The DELEG record creates a new delegation.
It is authoritative in the parent side of delegation and thus can be signed with DNSSEC.
This makes it possible to validate all delegation parameters, including those of future extensions.

The DELEGI record is an auxiliary record which does not create a delegation by itself but provides an optional layer of indirection.
It can be used to share the same delegation information across any number of zones.
DELEGI is treated like regular authoritative record.

The DELEG record can be used instead of or alongside a NS record to create a delegation.
The combination of DELEG+NS is fully compatible with old resolvers, facilitating the incremental rollout of this new method of delegation.

Future documents can use the extensibility mechanism for more advanced features like connecting to a name server with an encrypted transport.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in \\
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

Terminology regarding the Domain Name System comes from {{?BCP219}}, with addition terms defined here:

* legacy delegation: A delegation that is done with an NS RRset
* legacy response: A response that does use the DELEG protocol described in this document
* DELEG-aware: An authoritative server or resolver that follows the protocol defined in this document
* DELEG-unaware: An authoritative server or resolver that does not follow the protocol defined in this document

# DELEG and DELEGI Resource Record Types

The DELEG record, RR type TBD, and the DELEGI record, RR type TBD2 (different from that of DELEG), have the same wire and presentation formats,
but their semantics are different as described in a following section.
These records are defined for the IN class.

The record format is based on the extensible key=value list that was originally defined as "SvcParams" for the SVCB record type {{?RFC9460}}.
Unlike SVCB, the DELEG protocol does not have "SvcPriority" and "TargetName" fields.
The keys in the DELEG protocol are different than those used in SVCB.
To avoid confusion between the two protocols, the list of key=value parameters used by the DELEG protocol are called DelegInfos and will be tracked in their own IANA registy for Delegation Information.

The following rules are adapted from SVCB, but with changed names:

- The whole RDATA consists of a single list called "DelegInfos".
- DelegInfos consists of individual DelegInfo key=value pairs.
- Each DelegInfo pair has DelegInfoKey and a possibly optional DelegInfoValue.
- Each DelegInfo has a specified presentation format and wire encoding.
- Each DelegInfoKey has a presentation name and a registered key number.
- Each DelegInfoValue is in a format specific to its DelegInfoKey.

Implementations can reuse the same code to parse SvcParams and DelegInfos and only plug in a different list of key=value pairs for SVCB/HTTPS and DELEG/DELEGI record families.

The initial set of DelegInfoKeys and their formats are defined in {{nameserver-info}}.

## Presentation Format

The RDATA presentation format of the record consists of a single list, DelegInfos.

The DelegInfos presentation format is defined exactly the same as SvcParams in Section 2.1 of {{?RFC9460}}. The following rules are adapted from SVCB, but with changed names:

- DelegInfos is a whitespace-separated list with each DelegInfo consisting of a DelegInfoKey=DelegInfoValue pair, or a standalone DelegInfoKey.
- Individual element definitions are the same as {{?RFC9460}}:
  - The DelegInfo syntax is the same as SvcParam, but it references DelegInfo elements instead of SvcParam elements.
  - DelegInfoKey syntax is the same as SvcParamKey.
  - The syntax for unknown keys in Section 2.1 of {{?RFC9460}} applies.
  - The DelegInfoValue syntax is the same as SvcParamValue.
  - The rules from Appendix A of {{?RFC9460}} apply.
- All the requirements in Section 2.1 of {{?RFC9460}} apply.

TODO: SVCB allows an empty list. I guess it is not a problem, it is small and will simply be ignored?

## RDATA Wire Format

The RDATA portion of the RR types has variable length and entirely consists of a single "DelegInfos" element:

        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        /                         DelegInfos                            /
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

The format of the DelegInfos element is identical to the format of the SvcParams element defined in {{?RFC9460}} Section 2.2,
including the requirements for strictly increasing numeric order to keys and no key duplication allowed.

All the requirements in Section 2.2 of {{?RFC9460}} apply.

The DelegInfos element is a sequence of individual DelegInfo elements.
The wire format of an individual DelegInfo element is the same as for a SvcParam element,
but it references DelegInfo elements instead of SvcParam elements.
Note that though SVCB allows SvcParams to be absent, DELEG will always require at least one DelegInfo in DelegInfos.

                    +0 (MSB)                            +1 (LSB)
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    0:  |                          DelegInfoKey                         |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    2:  |                length of DelegInfoValue                       |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    4:  /                          DelegInfoValue ...                   /
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

The permissible lengths depend on the DelegInfoKey value.
Some future keys may have no DelegInfoValue, which would be indicated with an explicit 0 length.

## Overview of Differences between DELEG and DELEGI Semantics

The following is a brief summary of semantic differences between the DELEG and DELEGI types.

- DELEG creates a delegation for its owner name, similar to the NS RR type.
- DELEG and NS RR types can coexist at the same owner name.
- DELEG is authoritative in the parent zone of the delegated zone, similar to the DS RR type, and unlike the NS RR type.
- DELEG is signed by the parent zone of the delegated zone when using DNSSEC, similar to the DS RR type, and unlike the NS RR type.
- DELEG cannot be present at the apex of the delegated zone, similar to the DS RR type, and unlike the NS RR type.
- DELEG has special processing for being included in answers.

Conversely,

- DELEGI is like any normal RR and doesn't require any special processing.
- DELEGI does not create a delegation for its owner name.
- DELEGI cannot coexist at the same owner name with DELEG or NS RR types.
- DELEGI DNSSEC signing and record placement rules are the same as for any ordinary RR type.
- DELEGI is used as the target of the DELEG protocol's "include" mechanism, as described in section {{slist}}.

TODO: Add some introduction comparing how resolvers see legacy delegation (set of NS and A/AAAA records) and DELEG delegation (DELEG and DELEGI records with server-ipv4 and server-ipv6 keys)

# Use of DELEG Records

The DELEG RR set MAY contain multiple records.
A DELEG RR set MAY be present with or without NS or DS RR sets at the delegation point, though without NS records then DELEG-unaware software will not be able to resolve records in the the delegated zone.

DELEG RR sets MUST NOT appear at a zone's apex.
The erroneous inclusion of DELEG RR set at zone's apex will cause DNSSEC validation failures.
Servers MAY refuse to load such an invalid zone, similar to the DS RR type.

## Resolvers

### Signaling DELEG Support

There will be both DELEG and NS needed for delegation for a long time.
Both legacy delegation and the DELEG protocol enable recursive resolution.
This document defines a new EDNS flag to signal that a resolver is DELEG-aware and therefore does not need NS records or glue information in a referral response.

A resolver that is DELEG-aware MUST signal in queries that it supports the DELEG protocol by setting a bit in the OPT RR TTL as described in {{!RFC6891}}.
This bit referred to as the "DELEG" (DE) bit, expected to be assigned by IANA at Bit 2 in the EDNS Header Flags registry, as follows:

                +0 (MSB)                +1 (LSB)
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
      0: |   EXTENDED-RCODE      |       VERSION         |
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
      2: |DO|CO|DE|              Z                       |
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+

Setting the DE bit to one in a query indicates the resolver understands the DELEG semantics and does not need NS records to follow a referral.
The DE bit set to 0 indicates the resolver is not DELEG-aware, and therefore can only be served referrals with NS records and other data according to pre-DELEG specifications.

### Referral

The DELEG record creates a zone cut similar to the NS record.

If one or more DELEG records exist at a given delegation point, a DELEG-aware resolver MUST treat the name servers from those DELEG records as authoritative for the child zone.
In such a case, a DELEG-aware resolver MUST NOT use NS records for the zone if they are present, even if resolution using DELEG records has failed.
Such fallback from DELEG to NS would invalidate the security guarantees of the DELEG protocol.

If no DELEG record exists at a given delegation point, DELEG-aware resolvers MUST use NS records as specified by {{!RFC1034}}.
See {{dnssec-validators}} for more information about protection from downgrade attacks.

### Parent-side types, QTYPE=DELEG

Record types defined as authoritative on the parent side of zone cut, currently the DS and DELEG types, retain the same special handling as described in Section 2.6 of {{!RFC4035}}.

DELEG-unaware resolvers can get different types of answers for QTYPE=DELEG queries based on the configuration of the server, such as whether it is DELEG-aware and whether it also is authoritative for subdomains.
For example, a DELEG-unaware authoritative name server which has loaded DELEG records via the {{?RFC3597}} unknown types mechanism would answer with them only if there were no NS records at the owner name, and answer with an NS delegation otherwise.

### Algorithm for "Finding the Best Servers to Ask" {#finding-best}

This document updates instructions for finding the best servers to ask.
That information currently is covered in Section 5.3.3 of {{!RFC1034}} and Section 3.4.1 of {{!RFC6672}} with the text "2. Find the best servers to ask."
Section 3.1.4.1 of {{!RFC4035}} should have explicitly updated Section 5.3.3 of {{!RFC1034}} for the DS RR type, but failed to do so.
This document simply extends this existing behavior to DELEG RR type as well, and makes this special case explicit.

When a DELEG RR set exists for a delegation in a zone, DELEG-aware resolvers ignore any NS RR set for the delegated zone, whether from the parent or from the apex of the child.

Each delegation level can have a mixture of DELEG and NS RR types, and DELEG-aware resolvers MUST be able to follow chains of delegations which combines both types in arbitrary ways.

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

TODO: after the text below, refer back to this figure and show the order that a DELEG-aware resolver would take when there is a failure to find any good DELEG addresses at sub.sld.test, then any usable name servers at sub.sld.test, and then maybe a good DELEG record at test.

The terms SNAME and SLIST used here are defined in Section 5.3.2 of {{!RFC1034}}:

SNAME is the domain name we are searching for.

SLIST is a structure which describes the name servers and the zone which the resolver is currently trying to query.
Neither {{RFC1034}} nor this document define how a resolver uses SLIST; they only define how to populate it.

A DELEG-aware SLIST needs to be able to hold two types of information, delegations defined by NS records and delegations defined by DELEG records.
DELEG and NS delegations can create cyclic dependencies and/or lead to duplicate entries which point to the same server.
Resolvers need to enforce suitable limits to prevent damage even if someone has incorrectly configured some of the data used to create an SLIST.

This leads to a modifications of the description from earlier documents for DELEG-aware resolvers can find the best servers to ask.
That description becomes:

1. Determine deepest possible zone cut which can potentially hold the answer for a given (query name, type, class) combination:

    1. Start with SNAME equal to QNAME.

    1. If QTYPE is a type that is authoritative at the parent side of a zone cut (currently, DS or DELEG), remove the leftmost label from SNAME.
For example, if the QNAME is "test.example." and the QTYPE is DELEG or DS, set SNAME to "example.".

1. Look for locally-available DELEG and NS RR sets, starting at current SNAME.

    1. For a given SNAME, check for the existence of a DELEG RR set.
If it exists, the resolver MUST use its content to populate SLIST.
However, if the DELEG RR set is known to exist but is unusable (for example, if it is found in DNSSEC BAD cache), the resolver MUST NOT instead use an NS RR set; instead, the resolver MUST treat this case as if no servers were available.

    1. If a given SNAME is proven to not have a DELEG RR set but does have an NS RR set, the resolver MUST copy the NS RR set into SLIST.

    1. If SLIST is now populated, stop walking up the DNS tree.
However, if SLIST is not populated, remove the leftmost label from SNAME and go back to the first step, using the newly shortened SNAME.
Do not go back to the first step if doing so would exceed the amount of work that the resolver is configured to do when processing names; see {{too-much-work}}.

The rest of Step 2's description is not affected by this document.

TODO: Determine what to do about "QNAME=. QTYPE=DELEG" or "QNAME=. QTYPE=DS" queries, which by definition do not have answers in the DNS.

### Name Server Information for Delegation {#nameserver-info}

The DELEG and DELEGI records have four keys that describe information about name servers.
The purpose of this information is to populate the SLIST with IP addresses of the name servers for a zone.
The types of information defined in this document are:

* server-ipv4: a set of IPv4 addresses for name servers
* server-ipv6: a set of IPv6 addresses for name servers
* server-name: a set of hostnames of name servers; the addresses must be fetched
* include-delegi: a set of domain names that point to a DELEGI RR sets, which in turn have more information about the delegation

These keys MUST have a non-empty DelegInfoValue.

The presentation values for server-ipv4 and server-ipv6 are comma-separated list of one or more IP addresses of the appropriate family in standard textual format {{?RFC5952}} {{?RFC4001}}.
The wire formats for server-ipv4 and server-ipv6 are a sequence of IP addresses, in network byte order, for the respective address family.

The presentation values for server-name and include-delegi are sets of fully-qualified domain names, separated by commas.
The wire format for server-name and include-delegi are each a concatenated set of a wire-format domain names, where the root label provides the separation between names.
The names in the wire format MUST NOT be compressed.

TODO: Are they? Are we going to forbid normal zone file expansion where names without trailing . get current origin appended to them?

A DELEG or DELEGI record MUST have one, and only one, set of server information, chosen from the following:

- one server-ipv4 key
- one server-ipv6 key
- a pair consisting of one server-ipv4 key and one server-ipv6 key
- one server-name key
- one include-delegi key

This restriction only applies to a single DELEG or DELEGI record; a DELEG or DELEGI RR set can have records with different server information keys.

When using server-name, the addresses for all the names in the set must be fetched using normal DNS resolution.
This means the names in the value of the server-name key or the include-delegi key MUST NOT be inside the delegated domain.

With this initial DELEG specification, servers are still expected to be reached on the standard DNS port for both UDP and TCP, 53.  While a future specification is expected to address other transports using other ports, its eventual semantics are not covered here.

### Populating the SLIST from DELEG and DELEGI Records {#slist}

Each individual DELEG record inside a DELEG RR set, or each individual DELEGI record in a DELEGI RR set, can cause the addition of zero or more entries to SLIST.

A resolver processes each individual DELEG record within a DELEG RR set, or each individual DELEGI record in a DELEGI RR set, using the following steps:

1. If a record has more than one type of server information key (excluding the IPv4/IPV6 case), or has multiple server information keys of the same type, that record is malformed.
Stop processing this record.

1. If server-ipv4 and/or server-ipv6 keys are present inside the record, copy all of the address values into SLIST.
Stop processing this record.

1. If a server-name key is present in the record, resolve each name in the value into IPv4 and/or IPv6 addresses.
Copy these addresses into SLIST.
Stop processing this record.

1. If an include-delegi key is present in the record, resolve each name in the value using the DELEGI RR type.
Recursively apply the algorithm described in this section, after checking that the maximum loop count described in {{too-much-work}} has not been reached.

1. If none of the above applies, SLIST is not modified by this particular record.

A DELEG-aware resolver MAY implement lazy filling of SLIST, such as by deferring processing remaining records if SLIST already has what the resolver considers a sufficiently large pool of addresses to contact.

The order in which to try the servers in the final SLIST is outside the scope of this document.

## Authoritative Servers

DELEG-aware authoritative servers act differently when handling queries from DELEG-unaware clients (those with DE=0) than from DELEG-aware clients (those with DE=1).

The server MUST copy the value of the DE bit from the query into the response, to signal that it is a DELEG-aware server.

### DELEG-unaware Clients

DELEG-unaware clients do not use DELEG records for delegation.
When a DELEG-aware authoritative server responds to a DELEG-unaware client, any DELEG record in the response would not signal a zone cut, is not returned in referral responses, and is not considered authoritative on the parent side of a zone cut.
Because of this, DELEG-aware authoritative servers MUST answer as if they are DELEG-unaware to DELEG-unaware clients.
Please note this instruction does not affect DNSSEC signing, i.e. no special handling for NSEC type bitmap is necessary and DELEG RR type is accurately represented even for DELEG-unaware clients.

Two specific cases of DELEG-aware authoritative servers responding in a DELEG-unaware manner are described here.

#### DELEG-unaware Clients Requesting QTYPE=DELEG

From the perspective of DELEG-unaware clients, the DELEG RR type does not have special semantics and should behave like an old ordinary RR type, e.g. TXT.
Thus, queries with DE=0 and QTYPE=DELEG MUST result in a legacy response.
This would be a legacy response if there are NS records (the NS RRset in the AUTHORITY section, plus all relevant DNSSEC records), or the actual DELEG record if the owner name does not have NS records.

TODO: Should we have an example with auth having parent+child zone at the same time, and DE=0 QTYPE=DELEG query?

#### DELEG-unaware Clients with DELEG RRs Present but No NS RRs

DELEG-unaware clients might ask for a name which belongs to a zone delegated only with DELEG RRs (that is, without any NS RRs).
Such a zone is not resolvable for DELEG-unaware clients.
In this case, the DELEG record itself cannot create a zone cut, and the DELEG-aware authoritative server MUST return a legacy response.

An example of a legacy response using NXDOMAIN is in {{legacynxdomain}}.
This legacy response might be confusing for subdomains of zones which actually exist because DELEG-aware clients would get a different answer, namely a delegation.
TODO: debate response code in WG.

The authoritative server is RECOMMENDED to supplement these responses to DELEG-unaware resolvers with an {{!RFC8914}} Extended DNS Error using the (IANA-TBD) value "New Delegation Only" from the Extended DNS Error Codes registry.

### DELEG-aware Clients

When the client indicates that it is DELEG-aware by setting DE=1 in the query, DELEG-aware authoritative servers treat DELEG records as zone cuts, and the servers are authoritative on the parent side of the zone cut.
This new zone cut has priority over a legacy delegation.

#### DELEG-aware Clients Requesting QTYPE=DELEG

An explicit query for the DELEG RR type at a delegation point behaves much like query for the DS RR type: the server answers authoritatively from the parent zone.
All previous specifications for special handling queries with QTYPE=DS apply equally to QTYPE=DELEG.
In summary, the server either provides an authoritative DELEG RR set or declares its non-existence, with relevant DNSSEC proofs when requested and available.

#### Delegation with DELEG

If the delegation has a DELEG RR set, the authoritative server MUST put the DELEG RR set into the Authority section of the referral.
In this case, the server MUST NOT include the NS RR set in the Authority section.
Include the covering RRSIG following the normal DNSSEC procedure for answers with authoritative zone data.

Similarly, rules for DS RR set inclusion in referrals apply as specified by the DNSSEC protocol.

#### DELEG-aware Clients with NS RRs Present but No DELEG RRs

If the delegation does not have a DELEG RR set, the authoritative server MUST put the NS RR set into the authority section of the referral.
The absence of the DELEG RR set MUST be proven as specified by the DNSSEC protocol for authoritative data.

Similarly, rules for DS RR set inclusion into referrals apply as specified by the DNSSEC protocol.
Please note, in practice the same process and records are used to prove the non-existence of both DELEG and DS RR sets.

## DNSSEC Signers

The DELEG record is authoritative on the parent side of a zone cut and needs to be signed as such.
Existing rules from the DNSSEC specifications apply.
In summary: for DNSSEC signing, treat the DELEG RR type the same way as the DS RR type.

In order to protect validators from downgrade attacks this draft introduces a new DNSKEY flag ADT (Authoritative Delegation Types).
In zones which contain a DELEG RR set, this flag MUST be set to 1 in at least one of the DNSKEY records published in the zone.

## DNSSEC Validators {#dnssec-validators}

DELEG awareness introduces additional requirements on validators.

### Clarifications on Nonexistence Proofs

This document updates Section 4.1 of {{!RFC6840}} to include "NS or DELEG" types in the type bitmap as indication of a delegation point, and generalizes applicability of ancestor delegation proof to all RR types that are authoritative at the parent (that is, both DS and DELEG).
The text in that section is updated as follows:

An "ancestor delegation" NSEC RR (or NSEC3 RR) is one with:

-  the NS and/or DELEG bit set,

-  the Start of Authority (SOA) bit clear, and

-  a signer field that is shorter than the owner name of the NSEC RR,
   or the original owner name for the NSEC3 RR.

Ancestor delegation NSEC or NSEC3 RRs MUST NOT be used to assume
nonexistence of any RRs below that zone cut, which include all RRs at
that (original) owner name, other than types authoritative at the parent-side of a
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
Alternately, the validator must make sure that the delegation with an NS record is covered by an NSEC3
RR with the Opt-Out flag set.
Opt-Out is not applicable to DELEG RR type
because DELEG records are authoritative at the parent side of a zone cut in the same
way that DS RR types are.

### Referral downgrade protection

If the zone is DNSSEC-secure, and if any DNSKEY of the zone has the ADT flag set to 1, a DELEG-aware validator MUST prove the absence of a DELEG RR set in referral responses from this particular zone.

Without this check, an attacker could strip the DELEG RR set from a referral response and replace it with an unsigned (and potentially malicious) NS RR set.
A referral response with an unsigned NS and signed DS RR sets does not require additional proofs of nonexistence according to pre-DELEG DNSSEC specification, and it would have been accepted as a delegation without the DELEG RR set.

### Chaining

A Validating Stub Resolver that is DELEG-aware has to use a Security-Aware Resolver that is DELEG-aware and, if it is behind a forwarder, that forwarder has to be security-aware and DELEG-aware as well.

# Security Considerations

TODO: Add more here

## Preventing Over-work Attacks {#too-much-work}

Resolvers MUST prevent situations where accidental misconfiguration of zones or malicious attacks cause them to perform too much work when resolving.
This document describes two sets of actions that, if not controlled, could lead to over-work attacks:

- Names with many subdomains can cause walking up the tree to populate SLIST ({{finding-best}}) to be burdensome.
To prevent this, the resolver SHOULD NOT walk up more than %%TODO: come up with a number%% labels in order to contribute to SLIST.

- Long chains of include-delegi information ({{nameserver-info}}), and those with circular chains of include-delegi information, can be burdensome.
To prevent this, the resolver SHOULD NOT follow more than 3 include-delegi chains in an RR set when populating SLIST.
Note that include-delegi chains can have CNAME steps in them; in such a case, a CNAME step is counted the same as a DELEGI step when determining when to stop following a chain.

## Preventing Downgrade Attacks

TODO: this section is a bit redundant with "Referral Downgrade Protection" above; harmonize them.

During the rollout of the DELEG protocol, the operator of an authoritative server can upgrade the server software to be DELEG-aware before changing any DNS zones.
Such deployment should work and provide DELEG-aware clients with correct DELEG-aware answers.
However, the deployment will not be protected from downgrade attacks against the DELEG protocol.

To protect DNSSEC-secure DNS zones that use DELEG delegations, the delegating zone needs to have at least one DNSKEY with the ADT flag set to 1.
Failure to set this flag in a DNSKEY record in the zone allows an attacker to remove the DELEG RR set from referrals which contain the DS RR set, and replace the original signed DELEG RR set with an arbitrary unsigned NS set.
Doing so would be a downgrade from the strong protection offered by DNSSEC for DELEG.
That is, the DELEG protocol when used with upgraded DNSKEY records gives the same protection to DELEG that the zone's DS RR set has.
Without DELEG, there are no security guarantees for the NS RR set on the parent side of the zone cut.

Please note that a full DNSKEY rollover is not necessary to achieve the downgrade protection for DELEG.
Any single DNSKEY with the ADT flag set to 1 is sufficient; the zone can introduce an otherwise unused record into the DNSKEY RR set.
This DNSKEY RR can even use an unknown signing algorithm and zero-length key material to minimize size increase of the DNSKEY RR set.

# IANA Considerations

## Changes to Existing Registries

IANA is requested to allocate the DELEG RR and the DELEGI RR in the Resource Record (RR) TYPEs registry, with the meaning "enhanced delegation information" and referencing this document.

IANA is requested to assign a new bit in the DNSKEY RR Flags registry ({{!RFC4034}}) for the ADT bit (N), with the description "Authoritative Delegation Types" and referencing this document.
For compatibility reasons, we request the bit 14 to be used.
This value has been proven to work whereas bit 0 was proven to break in practical deployments (because of bugs).

IANA is requested to assign a bit from the EDNS Header Flags registry ({{!RFC6891}}), with the abbreviation DE, the description "DELEG enabled", and referencing this document.

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

To enable code reuse from SVCB parsers, the requirements for registered Name exactly copy requirements set by {{!RFC9460}} section 14.3.1:
The characters in the registered Name field entry MUST be lowercase alphanumeric or "-".
The name MUST NOT start with "key" or "invalid".

The registration policy for new entries is Expert Review ({{!RFC8126}}).
The designated expert MUST ensure that the reference is stable and publicly available and that it specifies how to convert the delegation information's presentation format to wire format.
The reference MAY be any individual's Internet-Draft or a document from any other source with similar assurances of stability and availability.
An entry MAY specify a reference of the form "Same as (other key name)" if it uses the same presentation and wire formats as an existing key.

This arrangement supports the development of new parameters while ensuring that zone files can be made interoperable.

### Initial Contents

The "DELEG Delegation Information" registry should be populated with the following initial registrations:

~~~
Number:  1
Name:  server-ipv4
Meaning:  A set of IPv4 addresses of name servers
Reference:  {{nameserver-info}} of this document
Change Controller:  IETF

Number:  2
Name:  server-ipv6
Meaning:  A set of IPv6 addresses of name servers
Reference:  {{nameserver-info}} of this document
Change Controller:  IETF

Number:  3
Name:  server-name
Meaning:  A set of hostnames of name servers
Reference:  {{nameserver-info}} of this document
Change Controller:  IETF

Number:  4
Name:  include-delegi
Meaning:  A set of domain names of DELEGI records
Reference:  {{nameserver-info}} of this document
Change Controller:  IETF

The registration for numbers 65280-65534 is reserved for private use.
The registration for number 65535 is reserved.
~~~

## Temporary Assignments

This section gives the values that can be used for interoperability testing before IANA makes permanent assignments.
The section will be removed when IANA makes permanent assignments.

  * DELEG RR type code is 61440
  * DELEGI RR type code is 65433
  * DELEG EDNS DE flag bit is 2
  * DNSKEY ADT (Authoritative Delegation Types) flag bit is 14

--- back

# Examples

## Root zone file
The following example shows an excerpt from a signed root zone.
It shows the delegation point for "example." and "test."

The "example." delegation has DELEG and NS records.
The "test." delegation has DELEG but no NS records.

TODO: Add examples that have server-name and include-delegi being sets of more than one name.

TODO: Examples that show DELEGI records in ns2.example.net and ns3.example.org.

    example.   DELEG server-ipv4=192.0.2.1 server-ipv6=2001:DB8::1
    example.   DELEG server-name=ns2.example.net.,ns3.example.org.
    example.   RRSIG DELEG 13 4 300 20260101000000 (
                            20250101000000 33333 . SigExampleDELEG/ )

    example.   NS    a.example.
    example.   NS    b.example.net.
    example.   NS    c.example.org.

    example.   DS    44444 13 2 ABCDEF01234567...
    example.   RRSIG DS 13 4 300 20260101000000 (
                            20250101000000 33333 . SigExampleDS )

    example.   NSEC  net. NS DS RRSIG NSEC DELEG
    example.   RRSIG NSEC 13 4 300 20260101000000 (
                            20250101000000 33333 . SigExampleNSEC+/ )

    ; unsigned glue for legacy (NS) delegation
    a.example. A     192.0.2.1
    a.example. AAAA  2001:DB8::1

The "test." delegation point has a DELEG record and no NS or DS records.

Please note:
This is an example of unnecessairly complicated setup to demonstrate capabilities of DELEG and DELEGI RR types.

    test.      DELEG server-ipv6=3fff::33
    test.      DELEG include-delegi=Acfg.example.org.
    test.      DELEG include-delegi=config2.example.net.
    test.      RRSIG DELEG 13 4 300 20260101000000 (
                            20250101000000 33333 . SigTestDELEG )

    test.      NSEC  . RRSIG NSEC DELEG
    test.      RRSIG NSEC 13 4 300 20260101000000 (
                            20250101000000 33333 . SigTestNSEC/ )

Delegations to org and net zones omitted for brevity.

## Example.org zone file
The following example shows an excerpt from an unsigned example.org zone.

    Acfg.example.org.    DELEGI server-ipv6=2001:DB8::6666
    Acfg.example.org.    DELEGI server-name=c.example.org.
    Acfg.example.org.    DELEGI include-delegi=subcfg.example.org.

    c.example.org.       AAAA   3fff::33

    subcfg.example.org.  DELEGI server-ipv4=203.0.113.1 server-ipv6=3fff::2

## Example.net zone file
The following example shows an excerpt from an unsigned example.net zone.

    b.example.net.       A      198.51.100.1

    config2.example.net. DELEGI server-name=b.example.org.


## Responses

The following sections show referral examples:

### DO bit clear, DE bit clear

#### Query for foo.example

    ;; Header: QR RCODE=NOERROR
    ;;

    ;; Question
    foo.example.  IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    example.   NS    a.example.
    example.   NS    b.example.net.
    example.   NS    c.example.org.

    ;; Additional
    a.example. A     192.0.2.1
    a.example. AAAA  2001:DB8::1

#### Query for foo.test

    ;; Header: QR AA RCODE=NXDOMAIN
    ;;

    ;; Question
    foo.test.   IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    .   SOA ...

    ;; Additional
    ;; OPT with Extended DNS Error: New Delegation Only


### DO bit set, DE bit clear

#### Query for foo.example


    ;; Header: QR DO RCODE=NOERROR
    ;;

    ;; Question
    foo.example.   IN MX

    ;; Answer
    ;; (empty)

    ;; Authority

    example.   NS    a.example.
    example.   NS    b.example.net.
    example.   NS    c.example.org.
    example.   DS    44444 13 2 ABCDEF01234567...
    example.   RRSIG DS 13 4 300 20260101000000 (
                            20250101000000 33333 . SigExampleDS )
    ;; Additional
    a.example. A     192.0.2.1
    a.example. AAAA  2001:DB8::1


#### Query for foo.test {#legacynxdomain}

    ;; Header: QR DO AA RCODE=NXDOMAIN
    ;;

    ;; Question
    foo.test.      IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    .          SOA ...
    .          RRSIG SOA ...
    test.      NSEC  . RRSIG NSEC DELEG
    test.      RRSIG NSEC 13 4 300 20260101000000 (
                            20250101000000 33333 . SigTestNSEC/ )

    ;; Additional
    ;; OPT with Extended DNS Error: New Delegation Only


### DO bit clear, DE bit set

#### Query for foo.example


    ;; Header: QR DE RCODE=NOERROR
    ;;

    ;; Question
    foo.example.  IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    example.   DELEG server-ipv4=192.0.2.1 server-ipv6=2001:DB8::1
    example.   DELEG server-name=ns2.example.net.,ns3.example.org.

    ;; Additional
    ;; (empty)

#### Query for foo.test

    ;; Header: QR AA RCODE=NOERROR
    ;;

    ;; Question
    foo.test.   IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    test.      DELEG server-ipv6=3fff::33
    test.      DELEG include-delegi=Acfg.example.org.
    test.      DELEG include-delegi=config2.example.net.

    ;; Additional
    ;; (empty)

Follow-up example in {{delegi-example}} explains ultimate meaning of this response.

### DO bit set, DE bit set

#### Query for foo.example

    ;; Header: QR DO DE RCODE=NOERROR
    ;;

    ;; Question
    foo.example.  IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    example.   DELEG server-ipv4=192.0.2.1 server-ipv6=2001:DB8::1
    example.   DELEG server-name=ns2.example.net.,ns3.example.org.
    example.   RRSIG DELEG 13 4 300 20260101000000 (
                            20250101000000 33333 . SigExampleDELEG/ )
    example.   DS    44444 13 2 ABCDEF01234567...
    example.   RRSIG DS 13 4 300 20260101000000 (
                            20250101000000 33333 . SigExampleDS )

    ;; Additional
    a.example. A     192.0.2.1
    a.example. AAAA  2001:DB8::1

#### Query for foo.test

    ;; Header: QR DO DE AA RCODE=NOERROR
    ;;

    ;; Question
    foo.test.      IN MX

    ;; Answer
    ;; (empty)

    ;; Authority
    test.      DELEG server-ipv6=3fff::33
    test.      DELEG include-delegi=Acfg.example.org.
    test.      DELEG include-delegi=config2.example.net.
    test.      RRSIG DELEG 13 4 300 20260101000000 (
                            20250101000000 33333 . SigTestDELEG )
    test.      NSEC  . RRSIG NSEC DELEG
    test.      RRSIG NSEC 13 4 300 20260101000000 (
                            20250101000000 33333 . SigTestNSEC/ )

    ;; Additional
    ;; (empty)

Follow-up example in {{delegi-example}} explains the ultimate meaning of this response.

## DELEGI Interpretation {#delegi-example}

In the examples above, the test. DELEG record uses indirection and points to other domain names with DELEGI, A, and AAAA records.
During resolution, a resolver will gradually build set of name servers to contact, as defined in {{slist}}.

To visualize end result of this process we represent full set of name servers in form of a 'virtual' DELEG RR set.

    test. DELEG server-ipv4=198.51.100.1
    test. DELEG server-ipv4=203.0.113.1
    test. DELEG server-ipv6=2001:DB8::6666
    test. DELEG server-ipv6=3fff::2
    ; IPv6 address 3fff::33 was de-duplicated (input RR sets listed it twice)
    test. DELEG server-ipv6=3fff::33

Implementations are free to use arbitrary representation for this data as it is not directly exposed via DNS protocol.

# Acknowledgments
{:numbered="false"}

This document is heavily based on past work done by Tim April in
{{?I-D.tapril-ns2}} and thus extends the thanks to the people helping on this which are:
John Levine, Erik Nygren, Jon Reed, Ben Kaduk, Mashooq Muhaimen, Jason Moreau, Jerrod Wiesman, Billy Tiemann, Gordon Marx and Brian Wellington.

Work on DELEG protocol has started at IETF 118 Hackaton.
Hackaton participants: Christian Elmerot, David Blacka, David Lawrence, Edward Lewis, Erik Nygren, George Michaelson, Jan Včelák, Klaus Darilion, Libor Peltan, Manu Bretelle, Peter van Dijk, Petr Špaček, Philip Homburg, Ralf Weber, Roy Arends, Shane Kerr, Shumon Huque, Vandan Adhvaryu, Vladimír Čunát, Andreas Schulze.

Other people joined the effort after the initial hackaton: Ben Schwartz, Bob Halley, Paul Hoffman, Miek Gieben ...
