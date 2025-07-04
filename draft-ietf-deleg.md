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

contributor:
-
    name: Christian Elmerot
    organization: Cloudflare
    email: christian@elmerot.se
-
    name: Edward Lewis
    organization: ICANN
    email: edward.lewis@icann.org
-
    name: Roy Arends
    organization: ICANN
    email: roy.arends@icann.org
-
    name: Shumon Huque
    organization: Salesforce
    email: shuque@gmail.com
-
    name: Klaus Darilion
    organization: nic.at
    email: klaus.darilion@nic.at
-
    name: Libor Peltan
    organization: CZ.nic
    email: libor.peltan@nic.cz
-
    name: Vladimír Čunát
    organization: CZ.nic
    email: vladimir.cunat@nic.cz
-
    name: Shane Kerr
    organization: NS1
    email: shane@time-travellers.org
-
    name: David Blacka
    organization: Verisign
    email: davidb@verisign.com
-
    name: George Michaelson
    organization: APNIC
    email: ggm@algebras.org
-
    name: Ben Schwartz
    organization: Meta
    email: bemasc@meta.com
-
    name: Jan Včelák
    organization: NS1
    email: jvcelak@ns1.com
-
    name: Peter van Dijk
    organization: PowerDNS
    email: peter.van.dijk@powerdns.com
-
    name: Philip Homburg
    organization: NLnet Labs
    email: philip@nlnetlabs.nl
-
    name: Erik Nygren
    organization: Akamai Technologies
    email: erik+ietf@nygren.org
-
    name: Vandan Adhvaryu
    organization: Team Internet
    email: vandan@adhvaryu.uk
-
    name: Manu Bretelle
    organization: Meta
    email: chantr4@gmail.com
-
    name: Bob Halley
    organization: Cloudflare
    email: bhalley@cloudflare.com

--- abstract
A delegation in the Domain Name System (DNS) is a mechanism that enables efficient and distributed management of the DNS namespace. It involves delegating authority over subdomains to specific DNS servers via NS records, allowing for a hierarchical structure and distributing the responsibility for maintaining DNS records.

An NS record contains the hostname of the nameserver for the delegated namespace. Any facilities of that nameserver must be discovered through other mechanisms. This document proposes a new extensible DNS record type, DELEG, for delegation of the authority for a domain. Future documents then can use this mechanism to use additional information about the delegated namespace and the capabilities of authoritative nameservers for the delegated namespace.
--- middle

# Introduction

In the Domain Name System {{!STD13}}, subdomains within the domain name hierarchy are indicated by delegations to servers which are authoritative for their portion of the namespace.  The DNS records that do this, called NS records, contain hostnames of nameservers, which resolve to addresses.  No other information is available to the resolver. It is limited to connect to the authoritative servers over UDP and TCP port 53. This limitation is a barrier for efficient introduction of new DNS technology.

The proposed DELEG record type remedies this problem by providing extensible parameters to indicate capabilities and additional information, such as glue that a resolver may use for the delegated authority. It is authoritative and thus signed in the parent side of the delegation making it possible to validate all delegation parameters (names and glue records) with DNSSEC.

This document only shows how DELEG can be used instead of or along side a NS record to create a delegation. Future documents can use the extensible mechanism for more advanced features like connecting to a name server with an encrypted transport.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in \\
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

Terminology regarding the Domain Name System comes from {{?BCP219}}, with addition terms defined here:

* legacy name servers: An authoritative server that does not support the DELEG record.
* legacy resolvers: A resolver that does not support the DELEG record.

# DELEG Record Type

The DELEG record uses a new resource record type, whose contents are identical to the SVCB record defined in {{?RFC9460}}. For extensions SVCB and DELEG use Service Parameter Keys (SvcParamKeys) and new SvcParamKeys that might be needed also will use the existing IANA Registry.

## Differences from SVCB

* DELEG can only have two priorities 0 indicating INCLUDE and 1 indicating a DIRECT delegation. These terms MUST be used in the presentation format of the DELEG record.
* INCLUDE and DIRECT delegation can be mixed within an RRSet.
* The final INCLUDE target is an SVCB record, though there can be further indirection using CNAME or AliasMode SVCB records.
* There can be multiple INCLUDE DELEG records, but further indirections through SVCB records have to comply with {{?RFC9460}} in that there can be only one AliasMode SVCB record per name.
* In order to not allow unbounded indirection of DELEG records the maximum number of indirections, CNAME or AliasMode SVCB is 4.
* The SVCB IPv4hint and IPv6hint parameters keep their key values of 4 and 6, but the presentation format with DELEG MUST be Glue4 and Glue6.
* Glue4 and Glue6 records when present MUST be used to connect to the delegated name server.
* The target of any DELEG record MUST NOT be '.'
* The target of a DELEG INCLUDE record MUST be outside of the delegated domain.
* The target of a DELEG DIRECT record MUST be a domain below the delegated domain.

# Use of DELEG record

A DELEG RRset MAY be present at a delegation point.  The DELEG RRset MAY contain multiple records. DELEG RRsets MUST NOT appear at a zone's apex.

A DELEG RRset MAY be present with or without NS or DS RRsets at the delegation point.

## Resolvers

### Signaling DELEG support

A resolver that is DELEG aware MUST signal its support by sending the DE bit when iterating.

This bit is referred to as the "DELEG" (DE) bit.  In the context of the EDNS0 OPT meta-RR, the DE bit is the TBD of the "extended RCODE and flags" portion of the EDNS0 OPT meta-RR, structured as follows (to be updated when assigned):

                +0 (MSB)                +1 (LSB)
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
      0: |   EXTENDED-RCODE      |       VERSION         |
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
      2: |DO|CO|DE|              Z                       |
         +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+

Setting the DE bit to one in a query indicates the resolver understands new DELEG semantics and does not need NS RR to follow a referral. The DE bit cleared (set to zero) indicates the resolver is unprepared to handle DELEG and hence can only be served NS, DS and glue in a delegation response.

Motivation: For a long time there will be both DELEG and NS needed for delegation. As both methods should be configured to get to a proper resolution it is not necessary to send both in a referral response. We therefore purpose an EDNS flag to be use similar to the DO Bit for DNSSEC to be used to signal that the sender understands DELEG and does not need NS or glue information in the referral.

### Referral

The DELEG record creates a zone cut similar to the NS record.

If a DELEG record exists on a given delegation point, all record types defined as authoritative in the child zone MUST be resolved using the name servers defined in the DELEG record. In such case resolver MUST NOT use NS records even if they happen to be present in cache, even if resolution using DELEG records have failed for some reason. Such fallback from DELEG to NS would invalidate security guarantees of DELEG protocol.

If no DELEG record exists on a given delegation point resolver MUST use NS records as specified by RFC1034.

### Parent-side types, QTYPE=DELEG

Record types defined as authoritative on the parent side of zone cut (currently DS and DELEG types) retain the same special handling as before, i.e. {{!RFC4035}} section 2.6 applies.

DELEG unaware recursive resolvers will not be able to determine correct NS set for QTYPE=DELEG queries. This is not a bug.

### Algorithm

This section updates instructions for step "2. Find the best servers to ask." of RFC1034 section 5.3.3 and {{!RFC6672}} section 3.4.1.

There are two important details:

- The algorithm description should explicitly describe RR types authoritative at the parent side of a zone cut. This is implied by {{!RFC4035}} section 3.1.4.1 for DS RR type but the text in the algorithm description was not updated. DELEG specification simply extends this existing behavior to DELEG RR type as well, and makes this special case explicit.

- When DELEG RRset exists, NS RRset is ignored on that particular zone cut by DELEG aware resolvers.

- DELEG and NS RR types can be used differently at each delegation level and resolver MUST be able follow chain of delegations which combines them in arbitrary ways.

Example of a valid delegation tree:

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

Terms SNAME and SLIST used in the rest of this section are defined in RFC 1034 section 5.3.2.:

SNAME           the domain name we are searching for.

SLIST           a structure which describes the name servers and the
                zone which the resolver is currently trying to query.

Modified description of Step 2. Find the best servers to ask follows:

Step 2 looks for a name server to ask for the required data.

First determine deepest possible zone cut which can potentially hold the answer for given (query name, type, class) combination:

- Start with SNAME equal to QNAME.
- If QTYPE is a type authoritative at the parent side of a zone cut (DS or DELEG), remove leftmost label from SNAME. E.g. if QNAME is Test.Example. and QTYPE is DELEG or DS, set SNAME to Example.
- TODO: what to do about ". DELEG" (or DS) query? That leaves zero labels left. That by definition does not exist ...

Further general strategy is to look for locally-available DELEG and NS RRsets, starting at current SNAME. If none are found, shorten SNAME by removing leftmost label and check again. This effectivelly finds the deepest known delegation point on the path between SNAME and the root:

- For given SNAME first check existence of DELEG RRset. If it exists, resolver MUST use it's content to populate SLIST. If the DELEG RRset is known to exist but is unusable (e.g. it is found in DNSSEC BAD cache), resolver MUST NOT fallback to NS RRset, even if it is locally available. Resolver MUST treat this case as if no servers were available/reachable.
- If a given SNAME is proven to not have a DELEG RRset but has NS RRset, resolver MUST copy it into SLIST.
- If SLIST is populated, terminate walk up the DNS tree.
- If SLIST is not populated, remove leftmost label from SNAME and inspect RR types for this new SNAME.

Rest of the Step 2's description is not affected by this document.

Please note the instructions to "Bound the amount of work" further down in the original text to apply. Suitable limits MUST be enforced to limit damage EVEN IF SOMEONE HAS INCORRECTLY CONFIGURED SOME DATA.

## Authoritative Servers
DELEG-aware authoritative servers act differently when handling queries from DELEG-unaware clients (those with DE=0) and queries from DELEG-aware clients (those with DE=1).

DNSSEC signing in the presence of DELEG records is unaffected.
That is, the presence or abscence of DELEG records is always reflected in the NSEC type bitmap, regardless of the type of query.

The server MUST copy the value of the DE bit from the query into the response.
(TODO: not really necessary protocol-wise, but might be nice for monitoring the deployment?)

### DELEG-unaware Clients

DELEG-unaware clients do not use DELEG records for delegation.
When a DELEG-aware authoritative server responds to a DELEG-unaware client, any DELEG RR in the response does not create zone cut, is not returned in referral responses, and is not considered authoritative on the parent side of a zone cut.
Because of this, DELEG-aware authoritative servers MUST answer as if they are DELEG-unaware except for two narrow cases described here.

#### DELEG-unaware Clients Requesting QTYPE=DELEG

In DELEG-unaware clients, records with the DELEG RRtype are not authoritative on the parent side.
Thus, queries with DE=0 and QTYPE=DELEG MUST result in a legacy referral response.

#### DELEG-unaware Clients with DELEG RRs Present but No NS RRs

DELEG-unaware clients might ask for a name which belongs to a zone delegated only with DELEG RRs (that is, without any NS RRs).
Such zone is, by definition, not resolvable for DELEG-unaware clients.
In this case the DELEG RR itself cannot create a zone cut, and the DELEG-aware authoritative server MUST return a response with an RCODE of NXDOMAIN (along with the DNSSEC proof of non-existance if the query had DO=1).
The authoritative server is RECOMMENDED to supplement DELEG unaware response with Extended DNS Error "New Delegation Only".
 
This result might be confusing for subdomains of zones which actually exist because DELEG-aware clients would get a different answer, namely a delegation.

TODO: debate if WG wants to do explicit SERVFAIL for this case instead of 'just' EDE.

### DELEG-aware Clients

When the client indicates that it is DELEG-aware by setting DE=1 in the query, DELEG-aware authoritative servers treat DELEG records as zone cuts, and the servers are authoritative on parent side of zone cut. This new zone cut has priority over legacy delegation with NS RRset.

#### DELEG-aware Clients Requesting QTYPE=DELEG

An explicit query for DELEG RR type at a delegation point behaves much like query for DS RR type: the server answers authortiatively from the parent zone. All previous specifications for special handling QTYPE=DS apply equally to QTYPE=DELEG. In summary, server either provides authoritative DELEG RRset or proves its non-existence.

#### Delegation with DELEG

If the delegation has a DELEG RRset, the authoritative server MUST put the DELEG RRset into the Authority section of the referral. In this case, the server MUST NOT include the NS RRset into the Authority section. Presence of the covering RRSIG follows the normal DNSSEC specification for answers with authoritative zone data.

Similarly, rules for DS RRset inclusion into referrals apply as specified by DNSSEC protocol.

#### DELEG-aware Clients with NS RRs Present but No DELEG RRs

If the delegation does not have a DELEG RRset, the authoritative server MUST put the NS RRset into the authority section of the referral. Absence of DELEG RRset must be proven as specified by DNSSEC protocol for authoritative data.

Similarly, rules for DS RRset inclusion into referrals apply as specified by the DNSSEC protocol. Please note in practice the same process and records are used to prove non-existence of DELEG and DS RRsets.

## DNSSEC Signers

The DELEG record is authoritative on the parent side of a zone cut and needs to be signed as such. Existing rules from DNSSEC specification apply. In summary: For DNSSEC signing, treat DELEG RR type the same way as DS RR type.

In order to protect validators from downgrade attacks this draft introduces a new DNSKEY flag ADT (Authoritative Delegation Types). In zones which contain a DELEG RRset this flag MUST be set to one in at least one DNSKEYs published in the zone.

## DNSSEC Validators

DELEG awareness introduces additional requirements on validators.

### Clarifications on Nonexistence Proofs

This document updates {{!RFC6840}} section 4.1 to include "NS or DELEG" types in type bitmap as indication of a delegation point and generalizes applicability of Ancestor delegation proof to all types authoritative at parent (i.e. DS and DELEG). Updated text follows:

An "ancestor delegation" NSEC RR (or NSEC3 RR) is one with:

-  the NS and/or DELEG bit set,

-  the Start of Authority (SOA) bit clear, and

-  a signer field that is shorter than the owner name of the NSEC RR,
   or the original owner name for the NSEC3 RR.

Ancestor delegation NSEC or NSEC3 RRs MUST NOT be used to assume
nonexistence of any RRs below that zone cut, which include all RRs at
that (original) owner name other types authoritative at the parent-side of
zone cut (DS and DELEG), and all RRs below that owner name regardless of
type.

### Insecure Delegation Proofs

This document updates {{!RFC6840}} section 4.4 to include secure DELEG support and explicitly states Opt-Out is not applicable to DELEG. Updated text follows:

Section 5.2 of {{!RFC4035}} specifies that a validator, when proving a
delegation is not secure, needs to check for the absence of the DS
and SOA bits in the NSEC (or NSEC3) type bitmap.  The validator also
MUST check for the presence of the NS or DELEG bit in the matching NSEC (or
NSEC3) RR (proving that there is, indeed, a delegation).
Alternately make sure that the delegation with NS record is covered by an NSEC3
RR with the Opt-Out flag set. Opt-Out is not applicable to DELEG RR type
because this it is authoritative at the parent side of a zone cut in the same
say as DS RR type.

### Referral downgrade protection

When DNSKEY flag ADT is set to one, the DELEG aware validator MUST prove absence of a DELEG RRset in referral responses from this zone.

Without this check, an attacker could strip DELEG RRset from a referral response and replace it with an unsigned (and potentially malicious) NS RRset. A referral response with an unsigned NS and signed DS RRsets does not require additional proofs of nonexistance according to pre-DELEG DNSSEC specification and it would have been accepted as a delegation without DELEG RRset.

### Chaining

A Validating Stub Resolver that is DELEG aware has to use a Security-Aware Resolver that is DELEG aware and if it is behind a forwarder this has to be security and DELEG aware as well.

# IANA Considerations

IANA is requested to allocate the DELEG RR in the Resource Record (RR) TYPEs registry, with the meaning of "enhanced delegation information" and referencing this document.

IANA is requested to assign a new bit in the DNSKEY RR Flags registry ({{!RFC4034}}) for the ADT bit (N), with the description "Authoritative Delegation Types" and referencing this document. For compatibility reasons we request the bit 14 to be used. This value has been proven to work whereas bit 0 was proven to break in practical deployments (because of bugs).

IANA is requested to assign a bit from the EDNS Header Flags registry ({{!RFC6891}}), with the abbreviation DE, the description "DELEG enabled" and referencing this document.

IANA is requested to assign a value from the Extended DNS Error Codes ({{!RFC8914}}), with the Purpose "New Delegation Only" and referencing this document.

For the RDATA parameters to a DELEG RR, the DNS Service Bindings (SVCB) registry ({{!RFC9460}}) is used.  This document requests no new assignments to that registry, though it is expected that future DELEG work will.

--- back

#  Examples

The following example shows an excerpt from a signed root zone. It shows the delegation point for "example." and "test."

The "example." delegation has DELEG and NS records. The "test." delegation has DELEG but no NS records.

    example.   300 IN DELEG DIRECT a.example. Glue4=192.0.2.1 (
                            Glue6=2001:DB8::1 )
    example.   300 IN DELEG INCLUDE ns2.example.net.
    example.   300 IN DELEG INCLUDE ns3.example.org.
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

    test.      300 IN DELEG INCLUDE ns2.example.net
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


### Query for foo.test

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
    example.   300 IN DELEG DIRECT a.example. Glue4=192.0.2.1 (
                            Glue6=2001:DB8::1 )
    example.   300 IN DELEG INCLUDE ns2.example.net.
    example.   300 IN DELEG INCLUDE ns3.example.org.

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
    test.      300 IN DELEG INCLUDE ns2.example.net

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

    example.   300 IN DELEG DIRECT a.example. Glue4=192.0.2.1 (
                            Glue6=2001:DB8::1 )
    example.   300 IN DELEG INCLUDE ns2.example.net.
    example.   300 IN DELEG INCLUDE ns3.example.org.
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
    test.      300 IN DELEG INCLUDE ns2.example.net.
    test.      300 IN RRSIG DELEG 13 4 300 20250214164848 (
                            20250207134348 21261 . 98Aac9f7A1Ac26Q..= )
    test.      300 IN NSEC  a.test. RRSIG NSEC DELEG
    test.      300 IN RRSIG NSEC 13 4 300  20250214164848 (
                            20250207134348 21261 . kj7YY5tr9h7UqlK..= )

    ;; Additional
    ;; (empty)


# Acknowledgments {:unnumbered}

This document is heavily based on past work done by Tim April in
{{?I-D.tapril-ns2}} and thus extends the thanks to the people helping on this which are:
John Levine, Erik Nygren, Jon Reed, Ben Kaduk, Mashooq Muhaimen, Jason Moreau, Jerrod Wiesman, Billy Tiemann, Gordon Marx and Brian Wellington.

# TODO

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

* Write a security considerations section
* Change the parameters form temporary to permanent once IANA assigned. Temporary use:
  * DELEG QType code is 65432
  * DELEG EDNS Flag Bit is 3
  * DELEG DNSKEY Flag Bit is 0

# Change Log

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

## since draft-wesplaap-deleg-00

* Clarified SVCB priority behavior

* Added section on differences to draft-homburg-deleg-incremental-deleg

## since draft-wesplaap-deleg-01

* Reorganised and streamlined the draft to the bare minimum for DELEG as an NS replacement
* Defined codepoints for temporary testing
* Added examples
