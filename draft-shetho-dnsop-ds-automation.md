---
title: Best Practice Recommendations for DS Automation
abbrev: DS Automation
docname: draft-shetho-dnsop-ds-automation-latest
date: {DATE}
stream: IETF
category: bcp

ipr: trust200902
area: Internet
workgroup: DNSOP Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: S. Sheng
    name: Steve Sheng
    organization: ICANN
    email: steve.sheng@gmail.com
 -
    ins: P. Thomassen
    name: Peter Thomassen
    org: deSEC, Secure Systems Engineering
    email: peter@desec.io

normative:

informative:

--- abstract

Enabling support for automatic acceptance of DS parameters directly from the Child DNS operator requires the registry/registrar to make a number of technical decisions.

Not all existing DS automation deployments have made the same choices with respect to these questions, leading to somewhat inconsistent behavior across TLDs. From the perspective of a registrant with domain names under various TLDs, this is unexpected and confusing.

We propose a set of recommendations for registries and registrars who wish to implement DS automation via CDS / CDNSKEY scanning.

--- middle

# Introduction

{{!RFC7344}}, {{!RFC8078}}, {{!RFC9615}} automate DNSSEC delegation trust maintenance by having the child publish CDS and/or CDNSKEY records which hold the prospective DS parameters. Parental Agents using these protocols have to make a number of technical decisions. These include:

  - Should DS automation involve the registrar or the registry, or both?
  - What kind of validity checks should be performed on DS parameters? Should those checks be performed upon acceptance, or also continuously when in place?
  - How do TTLs and caching impact DS provisioning? How important is timing in a child key change?
  - How are conflicts resolved when DS parameters are accepted through multiple channels (e.g. via a conventional channel and via automation)? In case both the registry and the registrar are automating DS updates, how to resolve potential collisions?
  - What is the relationship with other registration state parameters, such as registry or registrar locks?
  - Should a successful or rejected DS update trigger a notification to anyone?

Not all existing DS automation deployments have made the same choices with respect to these questions, leading to somewhat inconsistent behavior. From the perspective of a domain owner with domain names under various TLDs, this is unexpected and confusing. This document thus proposes a set of recommendations for registries and registrars who wish to implement CDS/CDNSKEY-based DS automation.

Readers are expected to be familiar with DNSSEC, including {{!RFC4033}}, {{!RFC4034}}, {{!RFC4035}}, {{?RFC6781}}, {{!RFC7344}}, {{!RFC7477}}, and {{?RFC8901}}.

## Requirements Notation

{::boilerplate bcp14}

## Terminology

Child zone:
: A DNS zone whose delegation is in the Parent zone

Child (DNS operator):
: DNS operator responsible for a Child zone.

DNS (Zone) Operator:
: The entity controlling the authoritative contents of (and delegations in) the zone file for a given domain name, and thus operationally responsible for maintaining the “purposeful” records in the zone file (such as IP address, MX, or CDS/CDNSKEY records).
The DNSSEC signing function, during whose execution additional records get added (such as RRSIG or NSEC(3) records), may be fulfilled by the same entity, or by one or more signing providers directly or indirectly appointed by the registrant. Zone contents are then made available for query by transferring them to the domain’s authoritative nameservers, which similarly may be operated by one or more entities. For the purpose of this definition, the details of how this is arranged are not relevant, as it is the content controlled by the DNS operator that eventually is served when queries are answered for the given zone.

Parent zone:
: A DNS zone that holds a delegation for a Child zone

Parent (DNS operator):
: The DNS operator responsible for a Parent zone, and thus involved with the maintenance of the delegation’s DNSSEC parameters (in particular, the acceptance of these parameters and the publication of corresponding DS records).

Registrant:
: The entity responsible for records associated with a particular domain name in a domain name registry (typically under a TLD such as .com or a SLD such as co.uk). The registrant maintains the records they are responsible for in the registry through the use of a registrar; the registrar acts as an intermediary for both the registry and the registrant.

Registrar:
: An entity through which registrants register domain names; the registrar performs this service by interacting directly with the registry in charge of the domain’s suffix. A registrar may also provide other services such as DNS service or web hosting for the registrant. In some cases, the registry directly offers registration services to the public, that is, the registry may also perform the registrar function.

Registry:
: The entity that controls the registry database and authoritative DNS service of domain names registered under a particular suffix, for example a top-level domain (TLD). A registry receives requests through an interface (usually EPP) from registrars to read, add, delete, or modify domain name registrations, and then makes the requested changes in the registry database and associated DNS zone. In some cases, the registry directly offers registration services to the public, that is, the registry may also perform the registrar function.

Reseller:
: An entity through which registrants register domain names if they don’t interact with the registrar directly. The reseller is part of a registrar’s retail channel and relays the registration request to the registrar. A reseller may also provide other services such as DNS service or web hosting for the registrant.

RRR Model:
: Refers to the registrant-registrar-registry interaction framework used for generic top level domains (gTLDs) as well as some country-code top-level domains (ccTLDs). In this model, registrants interact with a registrar to register and manage domain names. Registrars interact with the domain’s registry for the provision and management of domain names on the registrant’s behalf.

Otherwise, the terminology in this document is as defined in {{!RFC7344}}.

# Recommendations

## Validity Checks and Safety Measures

- What kind of validity checks should be performed on DS parameters?

- Should those checks be performed upon acceptance, or also continually when in place?

- How do TTLs and caching impact DS provisioning? How important is time in a child key change?

### Analysis

To maintain the basic resolution function, it is a reasonable strategy to avoid the deployment of bad DS record sets in the parent zone. It is therefore desirable for the Parent to verify that the DS record set resulting from an automated (or even manual) update does not break DNSSEC validation if deployed, and otherwise cancel the update. (The CDS/CDNSKEY specification captures this in RFC 7344 Section 4.1.)

To further reduce the impact of any misconfigured DS record set — be it from automated or from manual provisioning — the option to quickly roll back the delegation’s DNSSEC parameters is of great importance. This can be achieved by setting a comparatively low TTL on the DS record set in the parent domain, at the cost of reduced resiliency against nameserver unreachability due to the earlier expiration of cached records. To mitigate this availability risk from permanently lowered TTLs, it may be prudent to limit very short TTLs to a brief time period after a change to the DS configuration, during which rollbacks are most likely to occur.

One might expect low TTLs to cause increased load on the corresponding authoritative nameservers. However, recent research performed with 5-minute DS TTLs at ISC and a real-world deployment under .goog with (permanent) DS TTLs of 15 minutes have shown such TTLs to have negligible impact on the overall load of a registry’s authoritative nameserver infrastructure.

It thus appears advisable that registries reduce the DS record set’s TTL to a low value at the time it is updated, and restore the normal TTL value after a certain amount of time has passed. Pragmatic values for the reduced TTL value range between 5–15 minutes. The TTL reduction should be in effect at least until the previous DS record set has expired from caches, that is, the duration of the low-TTL period should exceed the normal TTL value. The routine re-signing of the DS RRset (usually after a few days) provides a convenient opportunity for resetting the TTL. The IETF REGEXT Working Group is finalizing a specification that allows such on-demand adjustments to the DS TTL.

It might look reasonable to also reduce the DS TTL “symmetrically on the other side”, that is, well in advance of an upcoming change. However, given the goal of enabling timely withdrawal of a botched DS RRset, it is not equally important for the previous (functional) DS RRset to be abandoned very quickly. Further, reducing the TTL ahead of time would require the Parent to anticipate when a DS change would occur, but the Parent has no a-priori knowledge of that. The TTL reduction could thus only occur once the Parent received information that a DS update is desired; doing so at this point, however, would require delaying the DS update by another full TTL (so that the long-TTL DS RRset has expired from all resolver caches). Introducing this additional delay does not seem justified. On the other hand, not reducing the old DS TTL ahead of time has the advantage of providing some resiliency against a potentially botched DS update, as clients using the previous DS RRset (cached with the normal TTL) would not see the updated DS RRset, and it could be withdrawn without them ever seeing it. Wrong DS RRsets will then only gradually impact clients, minimizing impact overall.

Finally, when accepting or rejecting a DS update, it may be helpful to notify relevant parties (such as the Child DNS operator or registrant). Such a reporting mechanism may also be used for notifications in case a problem is detected with an operational (unchanged) DS RRset, such as due to an incompatible change in the Child zone. For details, see Appendix B.4.

### Recommendation

tbd


## Multiple Submitting Parties

- How are conflicts resolved when DS parameters are accepted through multiple channels (e.g. via a conventional channel and via automation)?

- In case both the registry and the registrar are automating DS updates, how to resolve potential collisions?

### Analysis

There are multiple channels through which DS parameters can be accepted:

- The registry can, through some automated method, receive information about intended DS updates directly from the Child DNS Operator, and update the DS records;

- The registrar can, through some automated method, receive this information and relay the information to the registry;

- Registrars can obtain the information from the registrant via webform submission or other means and relay it to the registry.

ICANN SSAC has offered the following considerations in this context:

- When a manual DS update is performed without ensuring that the information informing automated DS maintenance requests (such as CDS/DNSKEY records) are updated as well, the delegation’s DS records may be reset to their previous state at the next run of the automation process. (Note that in case of CDNSKEY/CDS processing, these records will only remain valid when signed with keys authorized via the manually deployed DS records. Validity checks described in Appendix B.1 further ensure that updates do not break validation.)

  Some experts have proposed suspending DS automation once a manual DS update has occurred, e.g., until after the Child’s SOA serial is found to be updated. While appealing at first glance, automatic resumption of DS automation at SOA update time comes with a high risk of “timing surprises” – namely, when the SOA serial changes for reasons unrelated to the DNSSEC key configuration. Any arbitrary modification to the zone content would then suffice to trigger DS automation resumption, including the very common updating of DNSSEC signature validity timestamps (often a weekly routine). Furthermore, authoritative nameservers may have different serial offsets (e.g. in multi-provider setups), so that an apparent serial increase might actually be an illusion. It is therefore advised to not follow this practice.

  All things considered, it appears advisable that automated DS updates generally not be suspended when a manual DS update has occurred. An exception from this rule is when the entire DS record set was removed, in which case the registrant likely wants to disable DNSSEC on the delegation. DS automation should then be suspended so that automatic re-initialization (bootstrapping) does not occur. In all other cases, any properly authenticated DS updates received through an automated method should be considered as the current intent of the domain owner.

- Under special circumstances, it may be necessary to perform a manual DS update. One important example is when the authentication token or key used by the automated method is destroyed, in which case an automatic key rollover is impossible as the Child DNS operator can no longer authenticate the associated information. Another example is when several providers are involved, but one no longer cooperates (e.g., when removing a provider from a multi-provider setup). Disabling manual DS management interfaces is therefore strongly discouraged.

  Similarly, when the registrar is known to not support DNSSEC, it seems adequate for registries to not perform automated DS maintenance, in order to prevent situations in which a misconfigured delegation cannot be manually recovered by the registrant (for lack of a manual update interface at the registrar).

- When the RRR model is used, there is a potential for collision if both the registry and the registrar are automating DS updates. This collision risk disappears entirely when using “Generalized DNS Notifications” for triggering DS maintenance through one party’s designated endpoint (see Section 4.2.2), and can otherwise be mitigated if the registry and registrar agree that only one of them will automate DS updates.

  Note that no adverse consequences are expected if both parties perform DS automation. An exception is when during a key rollover, registry and registrar see different versions of the Child’s DS update requests, such as when CDS/CDNSKEY records are retrieved from different vantage points. This may lead to flapping of DS updates, which is not expected to be harmful as either DS RRset will allow for the validation function to continue to work. The effect subsides as the Child’s state eventually becomes consistent (expected to occur within one TTL), and can be further reduced by checking overlapping update requests for consistency.

  As a standard aspect of key rollovers (RFC 6781), the Child DNS operator is expected to monitor propagation of Child zone updates to all authoritative nameserver instances, and only proceed to the next step once replication has succeeded everywhere and the DS record set was subsequently updated (and in no case before the DS RRset’s TTL has passed). Any breakage resulting from improper timing on the Child side is outside of the Parent’s sphere of influence, and thus out of scope of DS automation considerations.

### Recommendation

tbd


## Registration Locks

- What is the relationship with other registration state parameters, such as EPP locks?

### Analysis

Registries and registrars can set various types of locks for domain registrations, usually upon the registrant’s request. Some locks clearly should have no impact on DS automation (such as clientDeleteLock / serverDeleteLock), while for other types of locks, in particular “update locks”, the interaction with automated DS maintenance is more interesting.

An overview of the various types of standardized locks and which impact on DS automation would appear plausible is given in Table 2. Some registries may offer additional types of locks whose meaning and set/unset mechanisms are defined according to a proprietary policy.

TODO: add table

The only lock types that may be recognized as having an impact on DS automation are the ones prohibiting “Update” operations (see first “Impact” column in the table).

When a serverUpdateProhibited lock (“registry lock”) is in place, there exists an expectation that this lock renders all otherwise updateable registration data immutable. It seems logical to extend this lock to DS updates as well.

The situation presents itself differently when a clientUpdateProhibited lock (“registrar lock”) is in place. While protecting against various types of accidental or malicious change (such as unintended changes through the registrar’s customer portal), this lock is much weaker than the registry lock, as its security model does not prevent the registrar’s (nor the registry’s) actions. This is because the clientUpdateProhibited lock can be removed by the registrar without an out-of-band interaction.

Under such a security model, no significant security benefit is gained by preventing automated DS maintenance based on a clientUpdateProhibited lock alone, while preventing it would make maintenance needlessly difficult. It therefore seems reasonable not to suspend automation when such a lock is present. The remainder of this section discusses this in detail.

In a world without DNSSEC, it was possible for a registration to be set up once, then locked and left alone (no maintenance required). With DNSSEC comes a change to this operational model: the DNSSEC configuration may have to be maintained in order to remain secure and operational. For example, the Child DNS operator may switch to another signing algorithm if the previous one is no longer deemed appropriate. Such changes entail updating the delegation’s DS records. If authenticated, these operations do not qualify as accidental or malicious change, but as normal and appropriate activity for securing ongoing operation.

To accommodate key or algorithm rollovers performed by the Child DNS operator, a means for maintaining DS records is needed. It is worth recalling that any authenticated DS update request (such as when published via CDS/CDNSKEY records on all nameservers) constitutes a legitimate request in the name of the registrant. Further, the resulting DS update is subject to the parent’s acceptance checks, and not applied when incompatible with the DNSSEC keys published in the child zone (see Appendix B.1).

Given that a clientUpdateProhibited lock protects against unintended changes (such as through the customer portal) while not preventing actions done by the registrar (or the registry) themself, the lock is not suitable for defending against actions performed illegitimately by the registrar or registry (e.g., due to compromise). Any attack on the registration data that is feasible in the presence of a registrar lock is also feasible regardless of whether DS maintenance is done automatically; in other words, DS automation is orthogonal to the attack vector that a registrar lock protects against. Considering that automated DS updates are required to be authenticated and validated for correctness, it thus appears that honoring such requests, while in the registrant’s interest, comes with no additional associated risk. (Automated DS maintenance may be disabled by requesting a registry lock, if so desired.)

Suspending automated DS maintenance at the registrar or registry therefore does not seem justified unless the registration data is locked at the registry level (e.g. when the registrant has explicitly requested a serverUpdateProhibited lock to be set). In particular, a registrar lock alone provides insufficient grounds for suspending DS maintenance.

Following this line of thought, some registries (e.g., .ch/.cz/.li) today perform automated DS maintenance even when an “update lock” is in place. Registries offering proprietary locks should carefully consider for each lock whether its scope warrants suspension.

The situation might appear less clear for DNSSEC bootstrapping, where automatic DS initialization is not required to maintain ongoing operation. In the absence of a registry lock, however, it is in the interest of security to enable DNSSEC when requested. The fact that a Child is requesting DS initialization through an authenticated, automated method expresses the registrant’s intent to have the delegation secured. There would seem to be little reason for the registrant to take (or order) these preparatory steps if not for their request to be acted upon.

Further, considering that some domains are put into clientUpdateProhibited lock by default, not honoring authenticated DS initialization requests needlessly imposes an additional burden of human intervention for unlocking and relocking the domain in order to facilitate DS provisioning after registration, in spite of the registrant already having expressed (to their Child DNS operator) the intent of securing their domain with DNSSEC. It therefore appears that DS initialization and rollovers should be treated the same way with respect to locks, and only be suspended while in serverUpdateProhibited lock status.

An analysis like the above of the security properties of registry and registrar locks with respect to automated DS maintenance is so far not contained in any specification document. The SSAC suggests that this aspect should be specified more clearly, such as by explicitly defining under which locks automated DS maintenance is permissible, or by defining a new “maintenance lock” status whose absence would indicate that automated DS maintenance is permissible.

### Recommendation

tbd


## Reporting

- Should a successful or rejected DS update trigger a notification to anyone?

### Analysis

In general, it cannot be assumed that the Child DNS operator is aware of the error conditions observed by the Parent. For example, they may not know that their CDS/CDNSKEY record contents are not acceptable to the Parent. Early reporting of rejected DS updates may help the Child DNS operator handle the situation in a timely manner.

Similarly, a delegation can break even without an update request to the DS record set. Such breakage may occur during key rollovers (RFC 6781) when the Child DNS operator proceeds to the next step early, without verifying that the delegation’s DS record set is in the expected state. For example, when an algorithm rollover is performed and the old signing algorithm is removed from the Child zone while the DS record set still references a key with that algorithm, validation errors may result.

The SSAC therefore suggests that entities performing automated DS maintenance should report on error conditions they encounter. Even when no update was requested, it may be worthwhile to occasionally check whether the current DS contents would be accepted today (see Appendix B.1), and communicate any failures without changing the published DS record set.

The following situations may be of particular interest for being reported:

  1. The DS record set has been provisioned

       a. manually, or

       b. automatically for initialization (DNSSEC bootstrapping), or

       c. for the first time after a manual change (so DS records are now under automatic maintenance, see Appendix B.2);

  2. The DS record set has been removed

       a. manually, or

       b. automatically via the automation protocol’s designated process;

  3. A pending DS update cannot be applied due to an error condition. There are various scenarios where an automated DS update might have been requested, but can’t be fulfilled. These include:

       a. The new DS record set would break validation/resolution or is not acceptable to the Parent for some other reason (see Appendix B.1);

       b. A lock prevents DS automation (see Appendix B.3);

  4. No DS update is due, but it was determined that the Child zone is no longer compatible with the existing DS record set (e.g. lack of DNSKEY algorithm).

For these reportworthy cases, the entity performing DS automation would be justified to attempt communicating the situation. Potential recipients are:

  - Child DNS operator, via the report channel contained in the “Generalized DNS Notification” that triggered the DS update, or via email (from SOA contact data);

  - Registrar (if DS automation is performed by the registry), via EPP (or similar channel);

  - Registrant (domain holder) or technical contact, via email.

For DS updates and deactivation (cases 1 and 2), it seems worthwhile to notify both the domain’s technical contact and the registrant. This will typically lead to one notification during normal operation of a domain.

For error conditions (cases 3 and 4), the registrant need not always be involved. It seems advisable to first notify the domain’s technical contact and the DNS operator serving the affected Child zone, and only if the problem persists for a prolonged amount of time (e.g., three days), notify the registrant.

When the RRR model is used and the registry performs DS automation, the registrar should always stay informed of any DS changes, e.g., via EPP (RFC 8590).

During regular maintenance of a secure delegation, standard updates of DS record sets are not of particular interest. In particular, if the delegation is already secure and a DS update has been applied automatically without any indication of irregularity, notifications to humans need not be triggered.

Further, the same condition should not be reported unnecessarily frequently to the same recipient (e.g., no more than twice in a row). For example, when CDS and CDNSKEY records are inconsistent and prevent DS initialization, the registrant may be notified twice. Additional notifications may be sent with some back-off mechanism (in increasing intervals).

The history of DS updates should be kept and, together with the currently active configuration, be made accessible to the registrant (or their designated party) through the customer portal available for domain management.

### Recommendation

tbd


## Consistency Considerations

- Are DS parameters best conveyed via DS format or DNSKEY format (or both)?

- How are conflicts resolved?

### Analysis

DS records can be generated from either DS-style or from DNSKEY-style input format. The former is identical to that of DS records (so can be taken verbatim), while for the latter, generation of a DS record involves computing a hash and other information based on the DNSKEY-format information.

Whether DS or DNSKEY format is ingested by the Parent depends on the Parent’s preference:

  - Conveying (and storing) parameters in DNSKEY format (such as via CDNSKEY records) allows the Parent to exert control over the choice of hash algorithms. The Parent may then unilaterally regenerate DS records with a different choice of hash algorithm(s) whenever deemed appropriate.

  - Conveying parameters in DS format (such as via CDS records) allows the Child DNS operator to control the hash digest type used in DS records, enabling the Child DNS operator to deploy (for example) experimental hash digests and removing the need for registry-side changes when new digest types become available.

The need to make a choice in the face of this dichotomy is not particular to DS automation: Even when DNSSEC parameters are relayed to the Parent through conventional channels, the Parent has to make some choice about which format(s) to accept.

Some registries have chosen to prefer DNSKEY-style input which seemingly comes with greater influence on the delegation’s security properties (in particular, the DS hash digest type). The SSAC notes that regardless of the choice of input format, the Parent cannot prevent the Child from following insecure cryptographic practices (such as insecure key storage, or using a key with insufficient entropy). Besides, as the DS format contains a field indicating the hash digest type, objectionable ones (such as those outlawed by RFC 8624) can still be rejected even when parameters are accepted as DS-style input, by inspecting that field.

The fact that more than one input type needs to be considered burdens both Child DNS operators and Parents with the need to consider how to handle this dichotomy. While the SSAC points out that this state of things is suboptimal, other community venues such as the IETF are better suited to address the dichotomy and evaluate the possibility of deprecating one of these mechanisms, with Parents transitioning to accepting input of the other type exclusively.

In the meantime, it seems worthwhile for both Child DNS operators and Parents implementing automated DS maintenance to act as to maximize interoperability. How that is best achieved depends on the automation mechanism in use. In any case, it will be helpful to record the registry’s practices in the DNSSEC Practice Statement (DPS).

For CDS/CDNSKEY-based automation, the following additional considerations are offered:

  - There exists no discovery protocol for the Parent’s input format preference. Recognizing that Child DNS operators are generally ignorant about which format the Parent prefers, they are encouraged to publish both CDNSKEY records as well as CDS records. The choice of hash digest type should follow current best practice, currently RFC 8624.

    The SSAC would like to emphasize that while publishing the same information in two different formats is not ideal, it seems to be the simpler choice (as opposed to requiring the DNS operator to bear the complexity cost of discovering which Parent prefers which format). In order to avoid operational issues, DNS operators should take great care to make sure that published records are consistent with each other, for example by synthesizing them, or programmatically updating them at once.

  - Parents, independently of their input format preference, are advised to require publication of both CDS and CDNSKEY records, and to enforce consistency between them, as determined by matching CDS and CDNSKEY records using hash digest algorithms whose support is required according to RFC 8624 Section 3.3. (For CDS records using another, unsupported hash digest, consistency is undefined and thus not required.)

    By rejecting the DS update if either type is found missing or inconsistent with the other, Child DNS operators are held responsible when publishing contradictory information. Registries can retain whatever benefit their choice carries for them, while at the same time facilitating a later revision of their choice. Similarly, this simplifies possible future deprecation of one of the two formats without breakage.

In order to further reduce the risk of wrongful DS deployment such as from nameserver hijacking or negligent multi-signer operation, it seems helpful to ensure that CDS and CDNSKEY responses are consistent across nameservers. This can be achieved by attempting to collect them from all authoritative nameservers listed in the delegation (one query per hostname suffices). When a key is referenced in a CDS or CDNSKEY record set returned by one nameserver, but is missing from another nameserver's answer, the Child's intent is unclear, and DS provisioning should be aborted. (TODO reference I-D.ietf-dnsop-cds-consistency).

### Recommendation

tbd


# IANA Considerations

This document has no IANA actions.


# Security Considerations

tbd


# Acknowledgments

SSAC DS Automation WP members.

--- back

# Change History (to be removed before publication)

* draft-shetho-dnsop-ds-automation-00

> Initial public draft.
