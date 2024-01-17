%%%
Title = "Best Practice for CDS Scanning"
abbrev = "cds-bestpractice"
docname = "@DOCNAME@"
category = "std"
ipr = "trust200902"
area = "Internet"
workgroup = "DNSOP Working Group"
date = @TODAY@

[seriesInfo]
name = "Internet-Draft"
value = "@DOCNAME@"
stream = "IETF"
status = "standard"

[[author]]
initials = "P."
surname = "Thomassen"
fullname = "Peter Thomassen"
organization = "SSE - Secure Systems Engineering GmbH"
[author.address]
 email = "peter.thomassen@securesystems.de"
[author.address.postal]
 street = "Hauptstraße 3"
 city = "Berlin"
 code = "10827"
 country = "Germany"
%%%


.# Abstract
Enabling support for automatic acceptance of DS parameters directly from the Child DNS operator requires the registry/registrar to make a number of technical decisions. These includes: (1) Should DS parameters be conveyed via CDS or CDNSKEY records, or both? (2) What kind of validity checks should be performed when ingesting DS parameters? (3) Should those checks be performed upon acceptance, or also continually when in place? (4) How are conflicts resolved when DS parameters are accepted through multiple channels (e.g. via EPP and via CDS/CDNSKEY)? (5) If both the registry and the registrar are automating DS updates, how to resolve potential collisions? (6) What is the relationship with other registration state parameters, such as EPP locks? (7) Should a successful or rejected DS update trigger a notification to anyone? (8) What is a suitable scanning interval? How can the cost of scanning be reduced?

Not all existing DS automation deployments have made the same choices with respect to these questions, leading to somewhat inconsistent behavior across TLDs. From the perspective of a registrant with domain names under various TLDs, this is unexpected and confusing.

We propose a set of best practices for registries and registrars who wish to implement DS automation via CDS / CDNSKEY scanning. 

{mainmatter}

# Introduction

[@!RFC7344] automates DNSSEC delegation trust maintenance by having the child publish CDS and/or CDNSKEY records which hold the prospective DS parameters.

Readers are expected to be familiar with DNSSEC, including [@!RFC4033], [@!RFC4034], [@!RFC4035], [@?RFC6781], [@!RFC7344], [@!RFC7477], and [@?RFC8901].

## Requirements Notation

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**", "**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in BCP 14 [@!RFC2119] [@!RFC8174] when, and only when, they appear in all
capitals, as shown here.

## Terminology

Child zone: 
: A DNS zone whose delegation is in the Parent zone

Child (DNS operator): 
: DNS operator responsible for a Child zone.

DS (Delegation Signer) record: 
: A DNS record located at a delegation in the Parent zone and containing the cryptographic fingerprint (hash digest, algorithm) of a DNSSEC public key (DNSKEY) whose private counterpart the Child uses (or intends to use) to sign its DNSKEY record set. The DS record set thus provides validation entry points to the Child zone and is signed by the Parent, thereby connecting the Child’s signing key(s) to the DNSSEC chain of trust that the Parent zone already has. There may be zero (DNSSEC off), one, or more DS records for any given delegation. 

DNSKEY record: 
: A DNS record containing a public DNSSEC validation key (matching a private DNSSEC signing key at the DNS operator). The key pair for a given DNSKEY record may be (operationally) constrained to sign/validate the zone’s DNSKEY record set only (“key-signing key”, KSK), thus authorizing additional keys to sign the rest of the zone’s content (“zone-signing keys”, ZSK). 

DNS (Zone) Operator: 
: The entity controlling the authoritative contents of (and delegations in) the zone file for a given domain name, and thus responsible for maintaining the “purposeful” records in the zone file (such as IP address, MX, or CDS/CDNSKEY records).

The DNSSEC signing function, during whose execution additional records get added (such as RRSIG or NSEC(3) records), may be fulfilled by the same entity, or by one or more signing providers directly or indirectly appointed by the registrant. Zone contents are then made available for query by transferring them to the domain’s authoritative nameservers, which similarly may be operated by one or more entities. For the purpose of this definition, the details of how this is arranged are not relevant, as it is the content controlled by the DNS operator that eventually is served when queries are answered for the given zone.

As a DNS query yields the response specified by the DNS operator, we thus say that the query is answered by the DNS operator. Other terms such as “DNS hosting provider”, “DNS provider”, “DNS service provider” are also often used to describe this concept.

DNSSEC Signer: 
: see DNS operator.

EPP:  
: The Extensible Provisioning Protocol (EPP), which is commonly used for communication of registration information between registries and registrars.  EPP is defined in [RFC5730].

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

TLS: 
: An authentication and encryption protocol widely implemented in browsers and Web servers. HTTP traffic transmitted using TLS is known as HTTPS. 

Otherwise, the terminology in this document is as defined in [@!RFC7344].

# Best Practices

## Validity Checks and Safety Measures

What kind of validity checks should be performed when ingesting DS parameters? Should those checks be performed upon acceptance, or also continually when in place?

Analysis: 
The CDS/CDNSKEY specification requires the Parent to verify that the resulting DS record set would not break DNSSEC validation if deployed, and otherwise cancel the update (RFC 7344 Section 4.1). This is a reasonable strategy to avoid DNSSEC validation breakage; bad DS record sets should not be deployed in the parent zone.
To further reduce the impact of any misconfigured DS record set — be it from automated or from manual provisioning — the option to quickly roll back the delegation’s DNSSEC parameters is of great importance. This can be achieved by setting a comparatively low TTL on the DS record set in the parent domain, at the cost of reduced resiliency against nameserver unreachability due to the earlier expiration of cached records. To mitigate this availability risk from permanently lowered TTLs, it may be prudent to limit very short TTLs to the time period shortly after a change to the DS configuration, during which rollbacks are most likely to occur.
One might expect low TTLs to cause increased load on the corresponding authoritative nameservers. However, recent research performed with 5-minute DS TTLs at ISC and a real-world deployment under .goog with (permanent) DS TTLs of 15 minutes have shown such TTLs to have negligible impact on the overall load of a registry’s authoritative nameserver infrastructure.

We recommend that registries reduce the DS record set’s TTL to a low value when it was updated, and restore the normal TTL value after a certain time has passed. Pragmatic values for the reduced TTL value range between 5–15 minutes. The TTL reduction should be in effect at least until the previous DS record set has expired from caches, that is, the duration of the low-TTL period should exceed the normal TTL value. The routine re-signing of the DS record set (usually after a few days) provides a convenient opportunity for resetting the TTL.

It might look reasonable to also reduce the DS TTL “symmetrically on the other side”, that is, well in advance of an upcoming change. However, the important aspect here is to enable timely withdrawal of a botched DS RRset; it is not equally important for a new DS RRset to take effect very quickly. Further, reducing the TTL ahead of time would require the Parent to anticipate when a DS change would occur, but the Parent has no a-priori knowledge of that. The TTL reduction could thus only occur after detecting the CDS/CDNSKEY, but doing so at this point in time would require delaying the DS update by another full TTL (so that the long-TTL DS RRset has expired from all resolver caches). Introducing this additional delay counteracts early use of the new DS RRset and thus does not seem justified. On the other hand, not reducing the DS TTL ahead of time has the advantage of providing some resiliency against a botched DS update, as clients using the previous DS RRset (cached with the normal TTL) would not see the updated DS RRset, and it could be withdrawn without them ever seeing it. Wrong DS RRsets will then only gradually impact clients, minimizing impact overall.

Finally, when accepting or rejecting a DS update, it may be helpful to notify relevant parties (such as the Child DNS operator or registrant). This reporting mechanism may also be used for notifications in case a problem is detected with an operational DS record set, such as due to an incompatible change in the Child zone. For details, see Appendix B.4.

Recommendations: 

Based on the above analysis, the best practice recommendation is: 

: Entities scanning for CDS/CDNSKEY records should verify that the resulting DS record set would not break DNSSEC validation if deployed, and otherwise cancel the update (RFC 7344 Section 4.1).

: Registries should reduce a DS record set’s TTL to a value between 5–15 minutes when it is updated, and restore the normal TTL value at the occasion of routinely re-signing the record set.

## Multiple Submitting Parties

How are conflicts resolved when DS parameters are accepted through multiple channels (e.g. via EPP and via CDS/CDNSKEY)? If both the registry and the registrar are automating DS updates, how to resolve potential collisions?

There are multiple channels through which DS parameters can be accepted:
: The registry can scan for CDS/CDNSKEY records and update the DS records;
: The registrar can do the scan and relay the information to the registry;
: Registrars can obtain the information from the registrant via webform submission or other means and relay it to the registry.

We would like to note several considerations in this context. When a manual DS update is performed without adjusting the existing CDS/CDNSKEY records accordingly, the delegation’s DS records may be reset to their previous state at the next run of the DS automation process. (Note that this can occur only when the CDS/CDNSKEY records are signed with keys authorized via the manually deployed DS records. Validity checks described in Appendix B.1 further ensure that CDS/CDNSKEY records, when applied, do not break validation.)

Some experts have proposed to suspend CDS/CDNSKEY processing once a manual DS update has occurred, until after the Child’s SOA serial is found to be updated. While appealing at first glance, automatic resumption of DS automation at SOA update time has a high risk of “timing surprises” – namely, when the SOA serial changes for some reason unrelated to the CDS/CDNSKEY records. Any arbitrary modification to the zone content will suffice to trigger DS automation resumption, including the very common updating of DNSSEC signature validity timestamps (often a weekly routine). Furthermore, authoritative nameservers may have different serial offsets (e.g. in multi-provider setups). It is therefore advised to not follow this practice.

It is further observed that when a zone is equipped with new keys and signatures, followed by a manual deployment of corresponding DS records, the existing CDS/CDNSKEY records – if left in place without modification – will not pass the Parent’s acceptance checks as they don’t match the new key material. Keeping DS automation active will thus not break the delegation, but can instead help correct the faulty CDS/CDNSKEY configuration via the error reporting mechanism described in Appendix B.4. Once the misconfiguration is fixed, DS automation will return to regular operation without further intervention.

Automated DS updates should thus generally not be suspended when a manual DS update has occurred. An exception from this rule is when the entire DS record set was removed, in which case the registrant likely wants to disable DNSSEC on the delegation. DS automation should thus be suspended so that automatic re-initialization (bootstrapping) does not occur. In all other cases, any CDS/CDNSKEY records present in the Child zone and properly signed should be considered as the current intent of the domain owner. (The presence of CDS/CDNSKEY records indicates that DS automation is desired.)

Under special circumstances, it may be necessary to perform a manual DS update. One important example is when the Child zone’s private key is destroyed, in which case a CDS/CDNSKEY-based key rollover is impossible because the Child DNS operator can no longer sign any record sets. Another example is when several providers are involved, but one no longer cooperates (e.g., when removing a provider from a multi-provider setup). Disabling manual DS management interfaces is therefore strongly discouraged.

Similarly, when the registrar is known to not support DNSSEC, registries should not perform automated DS maintenance, in order to prevent situations in which a misconfigured delegation cannot be recovered.
When the RRR model is used, there is a potential for collision if both the registry and the registrar are automating DS updates. This collision risk can be mitigated if the registry and registrar agree that only one of them will automate DS updates.

Note that no adverse consequences are expected if both parties perform DS automation. An exception is when during a key rollover, registry and registrar see different versions of the Child zone file from different vantage points, with different CDS/CDNSKEY record contents. This may lead to flapping of DS updates, which is expected to subside as replication eventually becomes consistent. This effect can be reduced by checking responses from authoritative nameservers for consistency (see Appendix B.5).

Further, as a standard aspect of key rollovers (RFC 6781), the Child DNS operator is expected to monitor propagation of Child zone updates to all authoritative nameserver instances, and only proceed to the next step once replication has succeeded everywhere and the DS record set was subsequently updated. Any breakage resulting from improper timing on the Child side is outside of the Parent’s sphere of influence, and thus out of scope of DS automation considerations.

Based on the above analysis, we recommend the best practice to be: 

: Registrars and (outside the RRR model) registries should provide a channel for manual DS maintenance in order to enable recovery when the Child has lost access to its signing key(s). It is also needed when a DNS operator does not support DS automation or refuses to cooperate.

: When DS updates are received through a manual or EPP interface, they should be executed immediately.

: Only when the entire DS record set has been removed through a manual or EPP submission should DS automation be suspended, in order to prevent accidental re-initialization of the DS record set when the registrant intended to disable DNSSEC.

: In all other cases where a non-empty DS record set is provisioned manually or via EPP (including after an earlier removal), DS automation should not (or no longer) be suspended. Any CDS/CDNSKEY records present in the Child zone and properly signed should in this case be considered as the current intent of the domain owner.

: In the RRR model, if the registry performs DS automation, the registry should notify the registrar of all DS updates (see also Recommendation 6.2.4d).

: Registries should not perform automated DS maintenance if it is known that the registrar does not support DNSSEC.

## Registration Locks

What is the relationship with other registration state parameters, such as EPP locks?

Registries and registrars can set various types of locks for domain registrations, usually upon the registrant’s request. Some locks clearly should have no impact on DS automation (such as clientDeleteLock / serverDeleteLock), while for other types of locks, in particular “update locks”, the interaction with automated DS maintenance is more interesting.

An overview of the various types of standardized EPP locks and their recommended impact on DS automation is given in Table 1. Some registries may offer additional types of locks whose meaning and set/unset mechanisms are defined according to a proprietary policy.

The only lock types that may be recognized as having an impact on DS automation are the ones prohibiting “Update” operations (see first “Impact” column in the table).

It is the SSAC’s position that when a serverUpdateProhibited lock (“registry lock”) is in place, all updates to the domain’s registration data, including any DS updates regardless of how they are submitted, should be disallowed. This is necessary in order to satisfy the expectation that this lock renders all otherwise updateable registration data immutable. (The only exception to this is the registry’s out-of-band process for removing this type of lock.)

The situation presents itself differently when a clientUpdateProhibited lock (“registrar lock”) is in place. While protecting against various types of accidental or malicious change (such as unintended changes through the registrar’s customer portal), this lock is much weaker than the registry lock, as its security model does not prevent the registrar’s (nor the registry’s) actions. This is because the clientUpdateProhibited lock can be removed by the registrar without an out-of-band interaction.

Under such a security model, no significant security benefit is gained by preventing automated DS maintenance based on a clientUpdateProhibited lock alone, while preventing it would make maintenance needlessly difficult. The SSAC therefore recommends not to suspend automation when such a lock is present. The remainder of this section discusses this in detail.

In a world without DNSSEC, it was possible for a registration to be set up once, then locked and left alone (no maintenance required). With DNSSEC comes a change to this operational model: the DNSSEC configuration may have to be maintained in order to remain secure and operational. For example, the Child DNS operator may switch to another signing algorithm if the previous one is no longer deemed appropriate. Such changes entail updating the delegation’s DS records. If authenticated by the Child DNS operator, these operations do not qualify as accidental or malicious change, but as normal and appropriate activity for securing ongoing operation.

To accommodate key or algorithm rollovers performed by the Child DNS operator, a means for maintaining DS records is needed. (Making this practical is the subject of this entire report.) It is worth recalling that a DS update request published via CDS/CDNSKEY records on all nameservers constitutes a legitimate request in the name of the registrant, underlined by the fact that it is signed. Further, the resulting DS update is subject to the parent’s acceptance checks, and not applied when incompatible with the DNSSEC keys published in the child zone (see Appendix B.1).

Given that a clientUpdateProhibited lock protects against unintended changes (such as through the customer portal) while not preventing actions done by the registrar (or the registry) themself, the lock is not suitable for defending against actions performed illegitimately by the registrar or registry (e.g., due to compromise). In other words, any attack on the registration data that is feasible in the presence of a registrar lock is also feasible regardless of whether DS maintenance is done automatically; automatic processing of CDS/CDNSKEY records is orthogonal to the attack vector that a registrar lock protects against. Considering that automated DS updates are authenticated and validated for correctness, it thus appears that honoring such requests comes with no additional associated risk, while in the registrant’s interest. (Automated DS maintenance may be disabled by requesting a registry lock, if so desired.)

Automated DS maintenance performed by either the registrar or the registry should therefore not be suspended unless the registration data is locked at the registry level (e.g. when the registrant has explicitly requested a serverUpdateProhibited lock to be set). In particular, a registrar lock alone should not prevent DS maintenance.

Following this line of thought, some registries (e.g., .ch/.cz/.li) today perform automated DS maintenance even when an “update lock” is set. Registries offering proprietary locks should carefully consider for each lock whether its scope warrants suspension.

The situation might appear less clear for DNSSEC bootstrapping, where automatic DS initialization is not required to maintain ongoing operation. In the absence of a registry lock, however, it is in the interest of security to enable DNSSEC when requested. The fact that a Child zone has been signed and equipped with CDS/CDNSKEY records requesting DS initialization clearly expresses the registrant’s intent to have the delegation secured, especially when the request is authenticated (as enabled by an IETF protocol draft expected to be finalized soon). It would hardly be understandable why the registrant would take (or order) these preparatory steps if not for their request to be acted upon.

Further, considering that some domains are put into clientUpdateProhibited lock by default, not honoring authenticated DS initialization requests needlessly imposes an additional burden of human intervention for unlocking and relocking the domain in order to facilitate DS provisioning after registration, in spite of the registrant already having expressed their intent of securing their domain with DNSSEC to their Child DNS operator. The SSAC therefore holds that DS initialization and rollovers should be treated the same way with respect to locks, and only be suspended while in serverUpdateProhibited lock status.

An analysis like the above of the security properties of registry and registrar locks with respect to automated DS maintenance is so far not contained in any specification document. The SSAC suggests that this aspect should be specified more clearly, such as by explicitly defining under which locks automated DS maintenance is permissible, or by defining a new “maintenance lock” status whose absence would indicate that automated DS maintenance is permissible.

While this report is concerned with DS management using CDS/CDNSKEY records, the SSAC would like to comment on the related matter of delegation NS record management. NS record updates can be automated using a mechanism similar to DS automation, via so-called CSYNC records (RFC 7477). This technique allows transferring control of a domain’s DNS service to another entity by effecting an NS change in the delegation. Its capabilities thus overlap with update as well as transfer actions, both of which can be locked via EPP. As NS automation is out of scope for this report, the SSAC at this time makes no recommendation about CSYNC processing. In particular, the below recommendations should not be construed to endorse automatic processing of CSYNC records in the presence of relevant EPP locks.

Based on the above analysis, the we recommend: 

: Automated DS maintenance should be suspended when a registry lock is set (in particular, EPP lock serverUpdateProhibited).

: To secure ongoing operations, automated DS maintenance should not be suspended based on a registrar lock alone (in particular, EPP lock clientUpdateProhibited).

: The wider community of registrars, registries, and standardization bodies like the IETF should consider specifying more clearly under which locks automated DS maintenance is permissible, including by potentially defining a new “maintenance lock” (such as serverMaintenanceProhibited) whose presence would disable automated DS maintenance.

## Reporting
Should a successful or rejected DS update trigger a notification to anyone?

In general, it cannot be assumed that the Child DNS operator is aware of the error conditions observed by the Parent. For example, they may not know that their CDS/CDNSKEY record contents are not acceptable to the Parent. Early reporting of rejected DS updates may help the Child DNS operator handle the situation in a timely manner.

Similarly, a delegation can break even when the CDS/CDNSKEY or DS record sets have not changed. Such breakage may occur during key rollovers (RFC 6781) when the Child DNS operator proceeds to the next step early, without verifying that the delegation’s DS record set is in the expected state. For example, when an algorithm rollover is performed and the old signing algorithm is removed from the Child zone while the DS record set still references a key with that algorithm, validation errors will result.

The SSAC therefore suggests that entities scanning for CDS/CDNSKEY records should report on error conditions they encounter. Even when the CDS/CDNSKEY record set has not changed, they should use the occasion and check whether it would be accepted today (see Appendix B.1), and communicate any failures without changing the published DS record set.

The following situations are of particular interest and considered worthy of being reported:
The DS record set has been provisioned
: manually, or
: automatically for initialization (DNSSEC bootstrapping), or
: for the first time after a manual change (so DS records are now in sync with CDS/CDNSKEY, see Appendix B.2);

The DS record set has been removed
: manually, or
: automatically via special CDS/CDNSKEY value;

A pending DS update cannot be applied due to an error condition. There are various scenarios where authenticated CDS/CDNSKEY records are available, but the associated DS update can’t be fulfilled. These include:
: The new DS record set would break validation/resolution or is not acceptable to the Parent for some other reason (see Appendix B.1);
: Some kind of lock prevents DS automation (see Appendix B.3)

No DS update is due, but in determining this it was found that the Child zone is no longer compatible with the existing DS record set (e.g. lack of DNSKEY algorithm).

For these reportworthy cases, the entity performing DS automation should attempt to communicate the situation. Potential recipients are:
: Registrant (domain holder), via email;
: Domain’s technical contact, via email;
: Registrar (if DS automation is performed by the registry), via EPP (or similar channel);
: DNS operators, via email (from SOA contact data).

For DS updates and deactivation (cases 1 and 2), it seems worthwhile to notify both the domain’s technical contact and the registrant. This will typically lead to one notification during normal operation of a domain.

For error conditions (cases 3 and 4), the registrant need not always be involved. It seems worthwhile to first notify the domain’s technical contact and the DNS operator serving the affected Child zone, and only if the problem persists for a prolonged amount of time (e.g., three days), notify the registrant.

When the RRR model is used and the registry performs DS automation, the registrar should always stay informed of any DS changes via EPP (RFC 8590) or a similar channel.

During regular maintenance of a secure delegation, standard updates of DS record sets are not of particular interest. In particular, if the delegation is already secure and a DS update has been applied automatically without any indication of irregularity (authenticated and valid CDS/CDNSKEY records), email notifications need not be triggered.

Further, the same condition should not be reported unnecessarily frequently to the same recipient (e.g., no more than twice in a row). For example, when CDS and CDNSKEY records are inconsistent and prevent DS initialization, the registrant may be notified twice. Additional notifications may be sent with some back-off mechanism (in increasing intervals).

The history of DS updates should be kept and, together with the currently active configuration, be made accessible to the registrant (or their designated party) through the customer portal available for domain management.

Based on the above analysis, we recommend: 
: For certain DS updates (see discussion in Appendix B.4) and for DS deactivation, both the domain’s technical contact and the registrant should be notified.
: For error conditions, the domain’s technical contact and the DNS operator serving the affected Child zone should be first notified, and only if the problem persists for a prolonged amount of time (e.g., three days), the registrant should be notified.
: These notifications should be done via email. The same condition should not be reported unnecessarily frequently to the same recipient.
: In the RRR model, if the registry performs DS automation, the registry should inform the registrar of any DS changes via EPP (RFC 8590) or a similar channel.
: The currently active DS configuration as well as the history of DS updates should be made accessible to the registrant (or their designated party) through the customer portal available for domain management.

## Consistency Considerations

Are DS parameters best conveyed via CDS or CDNSKEY records, or both?
How are conflicts resolved when both are present?
How are conflicts resolved when CDS or CDNSKEY records differ across nameservers?

Analysis: 
DS records can be generated from either CDS records or from CDNSKEY records. The former are in a format identical to that of DS records (so their content can be taken verbatim), while the latter are in the same format as DNSKEY records (and generation of a DS record involves computing a hash and other information based on the record content).
Whether CDS or CDNSKEY is ingested by the Parent depends on the Parent’s preference:
Conveying (and storing) parameters in DNSKEY format (such as via CDNSKEY records) allows the Parent to exert control over the choice of hash algorithms. The Parent may then unilaterally regenerate DS records with a different choice of hash algorithm(s) whenever deemed appropriate.
Conveying parameters in DS format (such as via CDS records) allows the Child DNS operator to control the hash digest type used in computing DS records, enabling the Child DNS operator to deploy (for example) experimental hash digests and removing the need for registry-side changes when new digest types become available.
Note that the need to make a choice in the face of this dichotomy is not particular to DS automation: Even when DNSSEC parameters are relayed to the Parent through conventional channels, the Parent has to make some choice about which format(s) to accept.
Some registries have chosen to prefer DNSKEY-style input which seemingly comes with greater influence on the delegation’s security properties (in particular, the DS hash digest type). The SSAC notes that regardless of the choice of input format, the Parent cannot prevent the Child from following insecure cryptographic practices (such as insecure key storage, or using a key that lacks sufficient entropy). Besides, blatantly insecure hash digest types (based on RFC 8624) can still be rejected even when parameters are accepted as DS-style input.
The fact that more than one input type is currently specified burdens both the Child DNS operators and Parents with the need to consider how to handle this dichotomy. While the SSAC points out that this state of things is suboptimal, other community venues such as the IETF are better suited to address the dichotomy and evaluate the possibility of deprecating one of these mechanisms, with Parents transitioning to accepting input of the other type exclusively.
In the meantime, both the Child DNS operator and the Parent implementing automated DS maintenance should act as to maximize interoperability. Besides documenting the registry’s practices in the DNSSEC Practice Statement (DPS), this means that:
There exists no discovery protocol for the Parent’s input format preference. Recognizing that Child DNS operators are generally ignorant about which format the Parent prefers, they are encouraged to publish both CDNSKEY records as well as CDS records. The choice of hash digest type should follow current best practice, currently RFC 8624.
The SSAC would like to emphasize that while publishing the same information in two different formats is not ideal, it seems to be the simpler choice (as opposed to requiring the DNS operator to bear the cost of discovering which Parent prefers which format). In order to avoid operational issues, DNS operators should take great care to make sure that published records are consistent with each other, for example by programmatically updating them at once.
Parents, independent of their input format preference, are advised to require publication of both CDS and CDNSKEY records, and to enforce consistency between them, as determined by matching CDS and CDNSKEY records using hash digest algorithms whose support is required according to RFC 8624 Section 3.3. (For CDS records using another, unsupported hash digest, consistency is undefined and thus not required.)
By rejecting the DS update if either type is found missing or inconsistent with the other, Child DNS operators are held responsible for publishing contradicting information. Registries can retain whatever benefit their choice carries for them, while at the same time facilitating the possibility to later revise their choice. Similarly, this simplifies possible future deprecation of one of the two formats without breakage.
In order to further reduce the risk of wrongful DS deployment such as from nameserver hijacking or negligent multi-signer operation, it seems helpful to ensure that CDS and CDNSKEY responses are consistent across nameservers. This can be achieved by attempting to collect them from all authoritative nameservers listed in the delegation. (One query per hostname suffices; it is not necessary to query each address or even anycast instance.)
When a key is referenced in a CDS or CDNSKEY record set returned by one nameserver, but is missing from at least one other nameserver's answer, the Child's intent is unclear, and DS provisioning can be aborted.
Suggestions: 
Based on the above analysis, the SSAC suggests:
… DNS operators to publish both CDNSKEY records as well as CDS records, as also recommended in Section 5 of RFC 7344, and follow best practice for the choice of hash digest type, currently published in RFC 8624.
… Parents, independently of their choice for CDS or CDNSKEY, to require publication of both kinds of records, and not to proceed with updating the DS record set if one is shown to be missing or inconsistent with the other.
… entities scanning for CDS/CDNSKEY records to attempt to collect CDS and CDNSKEY responses from all authoritative nameservers in the delegation (one query per type and hostname) and, if not found consistent across nameservers, cancel the update.
… entities scanning for CDS/CDNSKEY records to verify that any published CDS and CDNSKEY records are consistent with each other, and otherwise cancel the update.


# IANA Considerations

This document has no IANA actions.


# Security Considerations

The level of rigor mandated by this document is needed to prevent
publication of half-baked DS or delegation NS RRsets (authorized only
under an insufficient subset of authoritative nameservers), ensuring
that an operator in a (functioning) multi-provider setup cannot
unilaterally modify the delegation (add or remove trust anchors or
nameservers).
This applies both when the setup is intentional and when it is
unintentional (such as in the case of lame delegation hijacking).

As a consequence, the delegation's records can only be modified when
there is consensus across operators, which is expected to reflect the
domain owner's intentions.
Both availability and integrity of the domain's DNS service benefit from
this policy.

In order to resolve situations in which consensus about child zone
contents cannot be reached (e.g. because one of the nameserver
providers is uncooperative), Parental Agents SHOULD continue to accept
DS and NS/glue update requests from the domain owner via an
authenticated out-of-band channel (such as EPP [@!RFC5730]),
irrespective of the rise of automated delegation maintenance.
Availability of such an interface also enables recovery from a situation
where the private key is no longer available for signing the CDS/CDNSKEY
or CSYNC records in the child zone.


# Acknowledgments

SSAC DS Automation WP members. 


{backmatter}


# Change History (to be removed before publication)

* draft-thomassen-dnsop-cds-bestpractice-00

> Initial public draft.
