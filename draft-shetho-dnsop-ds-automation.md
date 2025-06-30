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

v: 3
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: S. Sheng
    name: Steve Sheng
    email: steve.sheng@gmail.com
 -
    ins: P. Thomassen
    name: Peter Thomassen
    org: deSEC
    email: peter@desec.io

normative:
  DNSKEY-IANA:
    target: "https://www.iana.org/assignments/dns-sec-alg-numbers/dns-sec-alg-numbers.xml#dns-sec-alg-numbers-1"
    title: DNS Security Algorithm Numbers
    author:
      name: IANA
  DS-IANA:
    target: "http://www.iana.org/assignments/ds-rr-types"
    title: Delegation Signer (DS) Resource Record (RR) Type Digest Algorithms
    author:
      name: IANA

informative:
  LowTTL:
    target: https://indico.dns-oarc.net/event/47/contributions/1010/attachments/958/1811/DS%20and%20DNSKEY%20TTL%20experiment.pdf
    title: "DS and DNSKEY low TTL experiments"
    author:
      -
        name: Petr Špaček
        org: ISC
    date: 2023-09-06
    seriesinfo:
      at: DNS OARC 41
  EPPstatuscodes:
    target: https://www.icann.org/resources/pages/epp-status-codes-2014-06-16-en
    title: "EPP Status Codes | What Do They Mean, and Why Should I Know?"
    author:
      -
        name: Petr Špaček
        org: ISC
    date: 2023-09-06
  SAC126:
    target: https://itp.cdn.icann.org/en/files/security-and-stability-advisory-committee-ssac-reports/sac-126-16-08-2024-en.pdf
    title: "SAC126: DNSSEC Delegation Signer (DS) Record Automation"
    author:
      -
        name: ICANN Security and Stability Advisory Committee (SSAC)
    date: 2024-08-12


--- abstract

Enabling support for automatic acceptance of DS parameters from the Child DNS operator (via RFCs 7344, 8078, 9615) requires the parent operator, often a registry or registrar, to make a number of technical decisions. This document describes recommendations for new deployments of such DS automation.

--- middle

# Introduction

{{!RFC7344}}, {{!RFC8078}}, {{!RFC9615}} automate DNSSEC delegation trust maintenance by having the child publish CDS and/or CDNSKEY records which indicate the delegation's desired DNSSEC parameters ("DS automation").

Parental Agents using these protocols have to make a number of technical decisions relating to issues of validity checks, timing, error reporting, locks, etc. Additionally, when using the RRR model (as is common amongst top-level domains), both the registrar and the registry can effect parent-side changes to the delegation. In such a situation, additional questions arise.

Not all existing DS automation deployments have made the same choices with respect to these questions, leading to somewhat inconsistent behavior. From the perspective of a domain holder with domain names under various TLDs, this may be unexpected and confusing.

New deployments of DS automation therefore SHOULD follow the recommendations set out in this document, to achieve a more uniform treatment across suffixes and to minimize user surprise.

In {{overview}}, operational questions are first raised and directly answered with the corresponding recommendations. After this overview, {{analysis}} follows with analysis and related considerations.

Readers are expected to be familiar with DNSSEC {{!RFC9364}}{{!RFC9615}}{{!I-D.ietf-dnsop-generalized-notify}}. For terminology, see {{terminology}}.

## Requirements Notation

{::boilerplate bcp14}


# Overview of Recommendations {#overview}

## Validity Checks and Safety Measures {#validity}

This section provides recommendations to address the following questions:

- What kind of validity checks should be performed on DS parameters?
- Should these checks be performed upon acceptance, or also continually when in place?
- How do TTLs and caching impact DS provisioning? How important is timing in a child key change?

For the rationale informing the below recommendations, see the analysis in {{analysis_validity}}.

{:notoc}
{:unnumbered}
### Recommendations

1. Entities performing automated DS maintenance SHOULD verify

    {:type="a"}
    1. the consistency of DS update requests across all authoritative nameservers in the delegation (one query per type and hostname) {{!I-D.ietf-dnsop-cds-consistency}}, and

    2. that the resulting DS record set would not break DNSSEC validation if deployed,

   and cancel the update if the verifications do not succeed.

2. Parent operators (such as registries) SHOULD reduce a DS record set's TTL to a value between 5–15 minutes when the set of records is changed, and restore the normal TTL value at a later occasion (but not before the previous DS RRset's TTL has expired).

## Reporting and Transparency {#reporting}

This section provides recommendations to address the following question:

- Should a failed (or even successful) DS update trigger a notification to anyone?

For the rationale informing the below recommendations, see the analysis in {{analysis_reporting}}.

{:notoc}
{:unnumbered}
### Recommendations

1. For certain DS updates (see {{analysis_reporting (analysis)}}) and for DS deactivation, both the domain's technical contact and the registrant SHOULD be notified.

2. For error conditions, the domain's technical contact and the DNS operator serving the affected Child zone SHOULD be first notified. The registrant SHOULD NOT be notified unless the problem persists for a prolonged amount of time (e.g., three days).

3. Notifications to humans SHOULD be done via email. Child DNS operators SHOULD be notified using a report query {{!RFC9567}} to the agent domain as described in ({{!I-D.ietf-dnsop-generalized-notify, Section 4}}). The same condition SHOULD NOT be reported unnecessarily frequently to the same recipient.

4. In the RRR model, if the registry performs DS automation, the registry SHOULD inform the registrar of any DS record changes via the EPP Change Poll Extension {{!RFC8590}} or a similar channel.

5. The currently active DS configuration as well as the history of DS updates SHOULD be made accessible to the registrant (or their designated party) through the customer portal available for domain management.

## Registration Locks {#locks}

This section provides recommendations to address the following question:

- How does DS automation interact with other registration state parameters, such as EPP locks?

For the rationale informing the below recommendations, see the analysis in {{analysis_locks}}.

{:notoc}
{:unnumbered}
### Recommendations

1. Automated DS maintenance SHOULD be suspended when a registry lock is set (in particular, EPP lock serverUpdateProhibited).

2. To secure ongoing operations, automated DS maintenance SHOULD NOT be suspended based on a registrar lock alone (in particular, EPP lock clientUpdateProhibited).

## Multiple Submitting Parties {#multiple}

This section provides recommendations to address the following questions:

- How are conflicts resolved when DS parameters are accepted through multiple channels (e.g. via a conventional channel and via automation)?
- In case both the registry and the registrar are automating DS updates, how to resolve potential collisions?

For the rationale informing the below recommendations, see the analysis in {{analysis_multiple}}.

{:notoc}
{:unnumbered}
### Recommendations

1. Registries and registrars SHOULD provide a channel for manual DS maintenance in order to enable recovery when the Child has lost access to its signing key(s). This manual channel is also needed when a DNS operator does not support DS automation or refuses to cooperate.

2. When DS updates are received through a manual or EPP interface, they SHOULD be executed immediately.

3. Only when the entire DS record set has been removed through a manual or EPP submission SHOULD DS automation be suspended, in order to prevent accidental re-initialization of the DS record set when the registrant intended to disable DNSSEC.

4. In all other cases where a non-empty DS record set is provisioned manually or via EPP (including after an earlier removal), DS automation SHOULD NOT (or no longer) be suspended.

5. In the RRR model, if the registry performs DS automation, the registry SHOULD notify the registrar of all DS updates (see also Recommendation 4 under {{reporting}}).

6. In the RRR model, registries SHOULD NOT perform automated DS maintenance if it is known that the registrar does not support DNSSEC.

## CDS vs. CDNSKEY

This section provides recommendations to address the following question:

- Are parameters for DS automation best conveyed as CDNSKEY or CDS records, or both?

For the rationale informing the below recommendations, see the analysis in {{analysis_dichotomy}}.

{:notoc}
{:unnumbered}
### Recommendations

1. DNS operators SHOULD publish both CDNSKEY records as well as CDS records, and follow best practice for the choice of hash digest type {{DS-IANA}}.

2. Parents, independently of their preference for CDS or CDNSKEY, SHOULD require publication of both RRsets, and SHOULD NOT proceed with updating the DS RRset if one is found missing or inconsistent with the other.

3. Registries (or registrars) scanning for CDS/CDNSKEY records SHOULD verify that any published CDS and CDNSKEY records are consistent with each other, and otherwise cancel the update {{!I-D.ietf-dnsop-cds-consistency}}.


# Analysis {#analysis}

## Validity Checks and Safety Measures {#analysis_validity}

### Continuity of Resolution

To maintain the basic resolution function, it is important to avoid the deployment of flawed DS record sets in the parent zone. It is therefore desirable for the Parent to verify that the DS record set resulting from an automated (or even manual) update does not break DNSSEC validation if deployed, and otherwise cancel the update.

This is best done by

1. verifying that consistent CDS/CDNSKEY responses are served by all of the delegation's nameservers {{!I-D.ietf-dnsop-cds-consistency}};

2. verifying that the resulting DS RRset does not break the delegation if applied ({{?RFC7344, Section 4.1}}), i.e., that it provides at least one valid path for validators to use ({{?RFC6840, Section 5.11}}). This is the case if there is at least one DS record referencing a key that actually signs the child's DNSKEY RRset, where the digest type and signing algorithm are listed as mandatory ("MUST") in the "Implement for DNSSEC Validation" columns of the relevant IANA registries {{DS-IANA}} and {{DNSKEY-IANA}}.

TODO Should checks be done continually? (Why is that the parent's task?) Or on demand, e.g., on a no-op NOTIFY?
Even when no update was requested, it may be worthwhile to occasionally check whether the current DS contents would be accepted today (see {{validity}}), and communicate any failures without changing the published DS record set.

### TTLs and Caching

To further reduce the impact of any misconfigured DS record set — be it from automated or from manual provisioning — the option to quickly roll back the delegation's DNSSEC parameters is of great importance. This is achieved by setting a comparatively low TTL on the DS record set in the parent domain, at the cost of reduced resiliency against nameserver unreachability due to the earlier expiration of cached records. The availability risk can be mitigated by limiting such TTLs to a brief time period after a change to the DS configuration, during which rollbacks are most likely to occur.

Registries therefore should significantly lower the DS RRset's TTL for some time following an update. Pragmatic values for the reduced TTL value range between 5–15 minutes.  Such low TTLs might be expected to cause increased load on the corresponding authoritative nameservers; however, recent research has demonstrated them to have negligible impact on the overall load of a registry's authoritative nameserver infrastructure {{LowTTL}}.

The reduction should be in effect at least until the previous DS record set has expired from caches, that is, the period during which the low-TTL is applied should exceed the normal TTL value. The routine re-signing of the DS RRset (usually after a few days) provides a convenient opportunity for resetting the TTL. When using EPP, the server MAY advertise its TTL policy via the domain `<info>` command described in {{?I-D.ietf-regext-epp-ttl, Section 2.1.1.2}}.

While this approach enables quick rollbacks, timing of the desired DS update process itself is largely governed by the previous DS RRset's TTL, and therefore does not generally benefit from an overall speed-up. Note also that nothing is gained from first lowering the TTL of the old DS RRset: such an additional step would, in fact, require another wait period while resolver caches adjust. For the sake of completeless, there likewise is no point to increasing any DS TTL values beyond their normal value.


## Reporting and Transparency {#analysis_reporting}

When accepting or rejecting a DS update, it cannot be assumed that relevant parties are aware of what's happening. For example, a registrar may not know when an automatic DS update is performed by the registry. Similarly, a Child DNS operator may not be aware when their CDS/CDNSKEY RRsets are out of sync across nameservers, thus being ignored. Early reporting of such conditions helps involved parties to act appropriately and in a timely manner.

A delegation can break even without an update request to the DS record set. This may occur during key rollovers ({{?RFC6781, Section 4.1}}) when the Child DNS operator proceeds to the next step early, without verifying that the delegation's DS RRset is in the expected state. For example, when an algorithm rollover is performed and the old signing algorithm is removed from the Child zone before the new DS record is added, validation errors may result.
TODO Reduce fearmongering: find numbers, better example, or loosen up wording.

Entities performing automated DS maintenance should report on conditions they encounter. The following success situations may be of particular interest:

  1. {:#reporting-1} A DS RRset has been provisioned

       {:type="a"}
       1. {:#reporting-1a} manually;

       2. {:#reporting-1b} due to commencing DS automation (either via DNSSEC bootstrapping, or for the first time after a manual change; see {{multiple}});

       3. {:#reporting-1c} automatically, as an update to an existing DS RRset that had itself been automatically provisioned.

  2. {:#reporting-2} The DS RRset has been removed

       {:type="a"}
       1. manually;

       2. automatically, using a delete signal ({{!RFC8078, Section 4}}).

In addition, there are error conditions worthy of being reported:

{:start="3"}
  3. {:#reporting-3} A pending DS update cannot be applied due to an error condition. There are various scenarios where an automated DS update might have been requested, but can't be fulfilled. These include:

       {:type="a"}
       1. The new DS record set would break validation/resolution or is not acceptable to the Parent for some other reason (see {{validity}}).

       2. A lock prevents DS automation (see {{locks}}).

  4. {:#reporting-4} No DS update is due, but it was determined that the Child zone is no longer compatible with the existing DS record set (e.g., DS RRset only references non-existing keys).

For these reportworthy cases, the entity performing DS automation would be justified to attempt communicating the situation. Potential recipients are:

  - Child DNS operator, preferably by making a report query {{!RFC9567}} to the agent domain listed in the EDNS0 Report-Channel option of the DS update notification that triggered the DS update ({{!I-D.ietf-dnsop-generalized-notify, Section 4}}), or alternatively via email to the address contained in the child zone's SOA RNAME field (see {{!RFC1035, Sections 3.3.13 and 8}});

  - Registrar (if DS automation is performed by the registry), via EPP (or similar channel);

  - Registrant (domain holder, in non-technical language, such as "DNSSEC security for your domain has been enabled and will be maintained automatically") or technical contact, via email.

For manual updates ({{reporting-1a (case 1a)}}{: format="none"}), commencing DS automation ({{reporting-1b (case 1b)}}{: format="none"}), and deactivating DNSSEC ({{reporting-2 (case 2)}}{: format="none"}), it seems worthwhile to notify both the domain's technical contact and the registrant. This will typically lead to one notification during normal operation of a domain. ({{reporting-1c (Case 1c)}}{: format="none"}, the regular operation of automation, is not an interesting condition to report to a human.)

For error conditions (cases {{reporting-3 (3)}}{: format="none"} and {{reporting-4 (4)}}{: format="none"}), the registrant need not always be involved. It seems advisable to first notify the domain's technical contact and the DNS operator serving the affected Child zone, and only if the problem persists for a prolonged amount of time (e.g., three days), notify the registrant.

When the RRR model is used and the registry performs DS automation, the registrar should always stay informed of any DS record changes, e.g., via the EPP Change Poll Extension {{!RFC8590}}.

The same condition SHOULD NOT be reported unnecessarily frequently to the same recipient (e.g., no more than twice in a row). For example, when CDS and CDNSKEY records are inconsistent and prevent DS initialization, the registrant may be notified twice. Additional notifications may be sent with some back-off mechanism (in increasing intervals).

The history of DS updates SHOULD be kept and, together with the currently active configuration, be made accessible to the registrant (or their designated party) through the customer portal available for domain management.


## Registration Locks {#analysis_locks}

Registries and registrars can set various types of locks for domain registrations, usually upon the registrant's request. An overview of standardized locks is given in {{Section 2.3 of ?RFC5731}}. Some registries may offer additional types of locks whose meaning and set/unset mechanisms are defined according to a proprietary policy.

While some locks clearly should have no impact on DS automation (such as clientDeleteProhibited / serverDeleteProhibited), other types of locks, in particular "update locks", may be recognized as having an impact on DS automation.

### Registry vs. Registrar Lock

When a serverUpdateProhibited lock ("registry lock") is in place, there exists an expectation that this lock renders all otherwise updateable registration data immutable. It seems logical to extend this lock to DS updates as well.

The situation is different when a clientUpdateProhibited lock ("registrar lock") is in place. While protecting against various types of accidental or malicious change (such as unintended changes through the registrar's customer portal), this lock is much weaker than the registry lock, as its security model does not prevent the registrar's (nor the registry's) actions. This is because the clientUpdateProhibited lock can be removed by the registrar without an out-of-band interaction.

Under such a security model, no tangible security benefit is gained by preventing automated DS maintenance based on a clientUpdateProhibited lock alone, while preventing it would make maintenance needlessly difficult. It therefore seems reasonable not to suspend automation when such a lock is present.

### Detailed Rationale

Pre-DNSSEC, it was possible for a registration to be set up once, then locked and left alone (no maintenance required). With DNSSEC comes a change to this operational model: the DNSSEC configuration may have to be maintained in order to remain secure and operational. For example, the Child DNS operator may switch to another signing algorithm if the previous one is no longer deemed appropriate, or roll its SEP key for other reasons. Such changes entail updating the delegation's DS records. If authenticated, these operations do not qualify as accidental or malicious change, but as normal and appropriate activity for securing ongoing operation.

To accommodate key or algorithm rollovers performed by the Child DNS operator, a means for maintaining DS records is needed. It is worth recalling that any properly authenticated DS update request constitutes a legitimate request in the name of the registrant. Further, the resulting DS update is subject to the parent's acceptance checks, and not applied when incompatible with the DNSSEC keys published in the child zone (see {{validity}}).

Given that a clientUpdateProhibited lock protects against unintended changes (such as through the customer portal) while not preventing actions done by the registrar (or the registry) themself, the lock is not suitable for defending against actions performed illegitimately by the registrar or registry (e.g., due to compromise). Any attack on the registration data that is feasible in the presence of a registrar lock is also feasible regardless of whether DS maintenance is done automatically; in other words, DS automation is orthogonal to the attack vector that a registrar lock protects against. Considering that automated DS updates are required to be authenticated and validated for correctness, it thus appears that honoring such requests, while in the registrant's interest, comes with no additional associated risk. (Automated DS maintenance may be disabled by requesting a registry lock, if so desired.)

Suspending automated DS maintenance therefore does not seem justified unless the registration data is locked at the registry level (e.g. when the registrant has explicitly requested a serverUpdateProhibited lock to be set). A registrar lock alone provides insufficient grounds for suspending DS maintenance.

Following this line of thought, some registries (e.g., .ch/.cz/.li) today perform automated DS maintenance even when an "update lock" is in place. Registries offering proprietary locks should carefully consider for each lock whether its scope warrants suspension.

In case of a domain not yet secured with DNSSEC, automatic DS initialization is not required to maintain ongoing operation; it might, however, request DNSSEC bootstrapping. In the absence of a registry lock, it is then in the interest of security to enable DNSSEC as requested. The fact that a Child is requesting DS initialization through an authenticated, automated method {{!RFC9615}} expresses the registrant's intent to have the delegation secured. There would be little reason for the registrant to have the corresponding CDS/CDNSKEY records published if not for their request to be acted upon.

Further, some domains are put into clientUpdateProhibited lock by default. In such cases, not honoring authenticated DS initialization requests imposes an additional burden on the registrant, who has to unlock and relock the domain in order to facilitate DS provisioning after registration. This is a needless cost especially for large domain portfolios. It is also unexpected, given that the registrant already has expressed their intent to have the domain secured to their DNS operator who in turn has published CDS/CDNSKEY records. It therefore appears that DS initialization and rollovers should be treated the same way with respect to locks, and only be suspended while in serverUpdateProhibited lock status.


## Multiple Submitting Parties {#analysis_multiple}

In the RRR model, there are multiple channels through which DS parameters can be accepted:

- The registry can retrieve information about an intended DS update automatically from the Child DNS Operator and apply the update directly;

- The registrar can retrieve the same and relay it to the registry;

- Registrars or (less commonly) registries can obtain the information from the registrant performing a "manual update", such as via webform submission, and relay it to the registry.

There are several considerations in this context, as follows.

### Necessity of Manual Updates

Under special circumstances, it may be necessary to perform a manual DS update. One important example is when the key used by for authentication of DS updates is destroyed: in this case, an automatic key rollover is impossible as the Child DNS operator can no longer authenticate the associated information. Another example is when several providers are involved, but one no longer cooperates (e.g., when removing a provider from a multi-provider setup). Disabling manual DS management interfaces is therefore strongly discouraged.

Similarly, when the registrar is known to not support DNSSEC (or to lack a manual interface), it seems adequate for registries to not perform automated DS maintenance, in order to prevent situations in which a misconfigured delegation cannot be manually recovered by the registrant.

### Impact of Manual Updates: When to Suspend Automation

When a manual DS update is performed in the presence of CDS/CDNSKEY records referencing the previous DS RRset's keys, the delegation's DS records may be reset to their previous state at the next run of the automation process. This section discusses in which situations it is appropriate to suspend DS automation after a manual update.

One option is to suspend DS automation after a manual DS update, but only until a resumption signal is observed. In the past, it was proposed that seeing an updated SOA serial in the child zone may serve as a resumption signal. However, as any arbitrary modification of zone contents — including the regular updating of DNSSEC signature validity timestamps  — typically causes a change in SOA serial, resumption of DS automation after a serial change comes with a high risk of surprise. Additional issues arise if nameservers have different serial offsets (e.g., in a multi-provider setup). It is therefore advised to not follow this practice.

Note also that "automatic rollback" due to old CDS/CDNSKEY RRsets can only occur if they are signed with a key authorized by one of new DS records. Validity checks described in {{validity}} further ensure that updates do not break validation.

All in all:

- It appears advisable to generally not suspend DS automation when a manual DS update has occurred.

- An exception from this rule is when the entire DS record set was removed, in which case the registrant likely wants to disable DNSSEC for the domain. DS automation should then be suspended so that automatic re-initialization (bootstrapping) does not occur.

- In all other cases, any properly authenticated DS updates received, including through an automated method, are to be considered as the current intent of the domain holder.

These conclusions are re-stated in normative language in {{multiple}}.

### Concurrent Automatic Updates

When the RRR model is used, there is a potential for collision if both the registry and the registrar are automating DS provisioning by scanning the child for CDS/CDNSKEY records. No disruptive consequences are expected if both parties perform DS automation. An exception is when during a key rollover, registry and registrar see different versions of the Child's DS update requests, such as when CDS/CDNSKEY records are retrieved from different vantage points. Although unlikely due to Recommendation 1a of {{validity}}, this may lead to flapping of DS updates; however, it is not expected to be harmful as either DS RRset will allow for the validation function to continue to work, as ensured by Recommendation 1b of {{validity}}. The effect subsides as the Child's state eventually becomes consistent (roughly, within the child's replication delay); any flapping until then will be a minor nuisance only.

The issue disappears entirely when scanning is replaced by notifications that trigger DS maintenance through one party's designated endpoint {{!I-D.ietf-dnsop-generalized-notify}}, and can otherwise be mitigated if the registry and registrar agree that only one of them will perform scanning.

As a standard aspect of key rollovers (RFC 6781), the Child DNS operator is expected to monitor propagation of Child zone updates to all authoritative nameserver instances, and only proceed to the next step once replication has succeeded everywhere and the DS record set was subsequently updated (and in no case before the DS RRset's TTL has passed). Any breakage resulting from improper timing on the Child side is outside of the Parent's sphere of influence, and thus out of scope of DS automation considerations.


## CDS vs. CDNSKEY {#analysis_dichotomy}

DS records can be generated from information provided either in DS format (CDS) or in DNSKEY format (CDNSKEY). While the format of CDS records is identical to that of DS records (so the record data be taken verbatim), generation of a DS record from CDNSKEY information involves computing a hash.

Whether a Parent processes CDS or CDNSKEY records depends on their preference:

- Conveying (and storing) CDNSKEY information allows the Parent to control the choice of hash algorithms. The Parent may then unilaterally regenerate DS records with a different choice of hash algorithm(s) whenever deemed appropriate.

- Conveying CDS information allows the Child DNS operator to control the hash digest type used in DS records, enabling the Child DNS operator to deploy (for example) experimental hash digests and removing the need for registry-side changes when new digest types become available.

The need to make a choice in the face of this dichotomy is not specific to DS automation: even when DNSSEC parameters are relayed to the Parent through conventional channels, the Parent has to make some choice about which format(s) to accept.

Some registries have chosen to prefer DNSKEY-style input which seemingly comes with greater influence on the delegation's security properties (in particular, the DS hash digest type). It is noted that regardless of the choice of input format, the Parent cannot prevent the Child from following insecure cryptographic practices (such as insecure key storage, or using a key with insufficient entropy). Besides, as the DS format contains a field indicating the hash digest type, objectionable ones (such as those outlawed by {{DS-IANA}}) can still be rejected even when ingesting CDS records, by inspecting that field.

The fact that more than one input type needs to be considered burdens both Child DNS operators and Parents with the need to consider how to handle this dichotomy. Until this is addressed in an industry-wide manner and one of these mechanisms is deprecated in favor of the other, both Child DNS operators and Parents implementing automated DS maintenance should act as to maximize interoperability:

- As there exists no protocol for Child DNS Operators to discover a Parent's input format preference, it seems best to publish both CDNSKEY as well as CDS records, in line with {{!RFC7344, Section 5}}. The choice of hash digest type should follow current best practice {{DS-IANA}}.

- Parents, independently of their input format preference, are advised to require publication of both CDS and CDNSKEY records, and to enforce consistency between them, as determined by matching CDS and CDNSKEY records using hash digest algorithms whose support is mandatory {{DS-IANA}}. (Consistency of CDS records with optional or unsupported hash digest types is not required.)

Publishing the same information in two different formats is not ideal. Still, it is much less complex and costly than burdening the Child DNS operator with discovering each Parent's policy; also, it is very easily automated. Operators should ensure that published RRsets are consistent with each other.

By rejecting the DS update if either RRset is found missing or inconsistent with the other, Child DNS operators are held responsible when publishing contradictory information. At the same time, Parents can retain whatever benefit their policy choice carries for them, while facilitating a later revision of that choice. This approach also simplifies possible future deprecation of one of the two formats, as no coordination or implementation changes would be needed on the child side.


# IANA Considerations

This document has no IANA actions.


# Security Considerations

This document considers security aspects throughout, and has not separate considerations.


# Acknowledgments

The authors would like to thank the SSAC members who wrote the {{SAC126}} report on which this document is based.

In order of first contribution or review: Barbara Jantzen, Matt Pounsett, Matthijs Mekking, Ondřej Caletka

--- back

# Terminology

Unless defined in this section, the terminology in this document is as defined in {{!RFC7344}}.

Child zone:
: DNS zone whose delegation is in the Parent zone.

Child (DNS operator):
: DNS operator responsible for a Child zone.

DNS operator:
: The entity holding the zone's primary copy before it is signed. Typically a DNS hosting provider in the domain holder's name, it controls the authoritative contents and delegations in the zone, and is thus operationally responsible for maintaining the "purposeful" records in the zone file (such as IP address, MX, or CDS/CDNSKEY records).
The parties involved in other functions for the zone, like signing and serving, are not relevant for this definition.

Parent zone:
: DNS zone that holds a delegation for a Child zone.

Parent (DNS operator):
: The DNS operator responsible for a Parent zone, and thus involved with the maintenance of the delegation's DNSSEC parameters (in particular, the acceptance of these parameters and the publication of corresponding DS records).

Registrant:
: The entity responsible for records associated with a particular domain name in a domain name registry (typically under a TLD such as .com or an SLD such as co.uk). When the RRR model is used, the registrant maintains these records through a registrar who acts as an intermediary for both the registry and the registrant.

Registrar:
: An entity through which registrants register domain names; the registrar performs this service by interacting directly with the registry in charge of the domain's suffix. A registrar may also provide other services such as DNS service or web hosting for the registrant. In some cases, the registry directly offers registration services to the public, that is, the registry may also perform the registrar function.

Registry:
: The entity that controls the registry database and authoritative DNS service of domain names registered under a particular suffix, for example a top-level domain (TLD). A registry receives requests through an interface (usually EPP {{?RFC5730}}) from registrars to read, add, delete, or modify domain name registrations, and then makes the requested changes in the registry database and associated DNS zone.

RRR Model:
: The registrant-registrar-registry interaction framework, where registrants interact with a registrar to register and manage domain names, and registrars interact with the domain's registry for the provision and management of domain names on the registrant's behalf. This model is common amongst top-level domains.


# Approaches not pursued

## Validity Checks and Safety Measures

### TTLs and Caching

It is not necessary to equally reduce the old DS RRset's TTL before applying a change. If this were done, the rollover itself would have to be delayed without any apparent benefit. With the goal of enabling timely withdrawal of a botched DS RRset, it is not equally important for the previous (functional) DS RRset to be abandoned very quickly. In fact, not reducing the old DS TTL has the advantage of providing some resiliency against a botched DS update, as clients would continue to use the previous DS RRset according to its normal TTL, and the broken RRset could be withdrawn without some of them ever seeing it. Wrong DS RRsets will then only gradually impact clients, minimizing impact overall.


# Change History (to be removed before publication)

* draft-shetho-dnsop-ds-automation-00

> Initial public draft.
