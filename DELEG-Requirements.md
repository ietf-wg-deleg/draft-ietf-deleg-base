# DELEG Requirements

These requirements originally came from [I-D.ietf-deleg-requirements]. 

## Additional Terminology

These terms are used as part of the requirements. 

DELEG:
: A new method of DNS delegation, matching the requirements in this document but not presuming any particular mechanism, including previous specific proposals that used this name

zone operator:
: The person or organization responsible for the nameserver which serves the zone


## Requirements Framework


The requirements constraining any proposed changes to DNS delegations fall broadly into two categories.


\"Hard requirements\" are those that must be followed by a successful protocol {{?RFC5218}}, because violating them would present too much of an obstacle for broad adoption.  These will primarily be related to the way the existing Domain Name System functions at all levels.


\"Soft requirements\" are those that are desirable, but the absence of which does not intrinsically eliminate a design.  These will largely be descriptive of the problems that are trying to be addressed with a new method, or features that would ease adoption.


The context used here will be for the Domain Name System as it exists under the IANA root and the Registry/Registrar/Registrant model {{?BCP219}}, and some conditions will only be relevant there. While it is expected that any design which satisfies the requirements of put forth here would be broadly applicable for any uses of the DNS outside of this environment, such uses are not in scope.


### Hard Requirements

The following strictures are necessary in a new delegation design.


* H1. DELEG must not disrupt the existing registration model of domains.

* H2. DELEG must be backwards compatible with the existing ecosystem. Legacy zone data must function identically with both DELEG-aware and DELEG-unaware software. Nameserver (NS) records will continue to define the delegation of authority between a parent zone and a child zone exactly as they have.

* H3. DELEG must not negatively impact most DNS software.  This is intentionally a bit vague with regard to "most", because it can't be absolutely guaranteed for all possible DNS software on the network.  However, the DNS community should strive to test any proposed mechanism against a wide range of legacy software and come to a consensus as to what constitutes adherence to this requirement.

* H4. DELEG must be able to secure delegations with DNSSEC.

* H5. DELEG must support updates to delegation information with the same relative ease as currently exists with NS records.   Changes should take the same amount of time (eg, controlled by a DNS TTL) and allow a smooth transition between different operational platforms.

* H6. DELEG must be incrementally deployable and not require any sort of flag day of universal change.

* H7. DELEG must allow multiple independent operators to simultaneously serve a zone.

### Soft Requirements


The following items are the aspirational goals for this work, describing the features that are desired beyond what current NS-based delegations provide.


* S1. DELEG should facilitate the use of new DNS transport mechanisms, including those already defined by DNS-over-TLS (DoT {{?RFC7858}}), DNS-over-HTTPS (DoH {{?RFC8484}}), and DNS-over-QUIC (DoQ {{?RFC9520}}).  It should easily allow the adoption of new transport mechanisms.


* S2. DELEG should make clear all of the necessary details for contacting a service -- its protocol, port, and any other data that would be required to initiate a DNS query.


* S3. DELEG should minimize transaction cost in its usage.  This includes, but is not limited to, packet count, packet volume, and the amount of time it takes to resolve a query.


* S4. DELEG should simplify management of a zone's DNS service.


* S5. DELEG should allow for backward compatibility to the conventional NS-based delegation mechanism.  That is, a zone operator who wants to maintain a single source of truth of delegation information using DELEG should be able to easily have Do53 delegations derived from it.


* S6. DELEG should be extensible and allow for the easy incremental addition of new delegation features after initial deployment.


* S7. DELEG should be able to convey a security model for delegations stronger than currently exists with DNSSEC.


### Non-Requirements

Several potential areas of requirement have been raised that are being explicitly acknowledged here as not being in the scope of this higher level document.

* Whether NS records must continue to be the primary means by which resolutions happen.

* Whether information for a new delegation mechanism is stored in at the zone name or elsewhere in the domain name hierarchy.

* If a new delegation protocol is based on a DNS resource record, that record must not appear in both the parent and child with the same name and type.  The protocol should clearly describe how to handle an occurrence of that record appearing in the child.
