---
title: "Distributed Aggregation Protocol (DAP) Extensions for Improved Application of Differential Privacy"
abbrev: "DAP DP Extensions"
category: std

docname: draft-thomson-ppm-dap-dp-ext-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Privacy Preserving Measurement"
keyword:
 - decibels
 - replay
venue:
  group: "Privacy Preserving Measurement"
  type: "Working Group"
  mail: "ppm@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/ppm/"
  github: "martinthomson/dap-dp-ext"
  latest: "https://martinthomson.github.io/dap-dp-ext/draft-thomson-ppm-dap-dp-ext.html"

author:
  -
    fullname: "Martin Thomson"
    organization: Your Organization Here
    email: "mt@lowentropy.net"

normative:

informative:
  SITE:
    target: https://html.spec.whatwg.org/#site
    title: HTML - Living Standard
    date: 2021-01-26
    author:
      -
        org: WHATWG


--- abstract

The Distributed Aggregation Protocol (DAP) can be a key component
of a system that provides differentially-private guarantees
for participants.
Extensions to DAP are defined to support these guarantees.
This includes bindings of reports to specific options,
so that the aggregation service can better implement privacy budgeting
and replay protections.

--- middle

# Introduction

The Distributed Aggregation Protocol (DAP) {{!DAP=I-D.ietf-ppm-dap}}
can be used as part of a differentially-private system.

Differential privacy depends on being able to limit the contributions
from participants.
The basic mechanism that DAP uses to cap contributions is anti-replay.
Aggregators are responsible for ensuring that
the same report cannot be aggregated more than once.
An honest participant will contribute a limited number of reports
and can rely on at least one aggregator
preventing that report from being used multiple times.
(The threat model does not seek to protect the privacy of a dishonest participant.)

This basic anti-replay mechanism allows DAP
to provide caps on contributions.
The resulting system is somewhat inflexible,
which can limit the applicability of the protocol
outside of the narrowly-defined usage modes in the basic specification.

This document defines several upload extensions to DAP
that either enable greater flexibility
or help constrain the flexibility allowed by other options.

{:aside}
> TODO: It would make sense for the corresponding extensions to be added
> to {{?TASKPROV=I-D.ietf-ppm-dap-taskprov}}.
> However, that protocol does not include any provision for extensions.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Late Task Binding {#late-bind}

DAP presently requires that a client be aware of the task that it is contributing to.
The identity of the task is bound to each report
through the inclusion of the task ID in the call to the sharding function
of the VDAF ({{Section 5.1 of ?VDAF=I-D.irtf-cfrg-vdaf}}).

The late_binding upload extension
(codepoint 0xTBD)
signals to aggregators that a report was not bound to a specific task
when it was created.

Late task binding might be useful
when reports are collected by an intermediary.
The client that generates the report in this case
might be unaware
of how the report will ultimately be aggregated.
This allows the intermediary to defer the creation of a task
until it has determined the necessary parameters for the task.

When sharding and protecting reports,
the task ID is replaced with a the fixed, 32-byte sequence of
b13e8440f1cdb4da51eed3967e0a2652d27f5005bc35f751daf188b4b746708b
(in hex).

{:aside}
> This is the output from SHA-256 {{?SHA2=RFC6234}}
> when passed an ASCII-encoded {{?ASCII=RFC0020}} input of
> 'no task_id'.

Enforcing anti-replay for a report
that is not bound to a specific task
is challenging.
An aggregator cannot constrain its search for duplicate reports
to those that were submitted to the task.
This could greatly increase the cost of meeting anti-replay requirements.
The intent with this extension is that additional constraints,
such as one or more of the scoping extensions (see {{scoping}}),
will be used to make it more feasible
for an aggregator to comply with anti-replay requirements.


# Scoping Extensions {#scoping}

The DAP upload extensions in this section might be used to either
constrain the use of reports
for tasks that are configured with matching values
or group reports for the purposes of detecting duplicates.

Including additional scoping information can also
ensure that reports do not get reused
outside of their intended scope.

This section defines upload extensions that carry
requester identity ({{requester}})
and report partition ({{partition}}).


## Requester (Website) Identity {#requester}

Reports might be requested by
an entity that operates at lower trust level than
the entity that assembles the report.
The entity at the lower trust level
might not have access to the information necessary
to generate the report.

The requester_identity upload extension
(codepoint 0xTBD)
contains an encoding of the entity
that requested the report be created.

For example, an application could ask the operating system
to generate a report
using information that would normally be withheld from it.
Similarly, a website could ask a web browser to generate a report
based on otherwise secret information.
In either case,
the release of information for report
is conditional on it only being used
by a specific aggregation service
under terms that have been previously established
with the aggregators.
Binding the report to the identity of the requester
ensures that any use of the system can be accounted for
as coming from that requester.

The specific encoding used in this extension
will depend on the application.
However, the use of a globally-unique identifier,
such as an origin ({{?ORIGIN=RFC6454}})
or serialized site ({{SITE}}),
reduces the likelihood of name collisions.
A name collision might either allow two requesters
that share an aggregator
to share and reuse each others reports
(or perhaps to marginally increase the odds
of having reports spuriously detected duplicates).


## Report Partition {#partition}

This extension allows a client to bind a report
to an application-defined label.
This allows applications to partition reports
and have each partition managed separately.

The report_partition upload extension
(codepoint 0xTBD)
contains an application-defined sequence of bytes.

The use of this upload extension allows aggregators
to partition their state for tracking reports.
Duplicate reports only need to be tracked
across a matching partition,
for detecting duplicates within a task
or for detecting duplicates across tasks.

The selection of partition values might need
to be coordinated with aggregators.
If partitions are used by aggregators,
the amount of state the aggregator tracks
is increased by the number of partitions.
This represents an increase in total storage,
in exchange for reducing the scope
over which that storage needs to be consistent.

An aggregator could constrain the values
that are accepted for this extension,
rejecting reports that lack the extension
or have disallowed values.


# Privacy Budget Consumption {#budget}

The gathering of reports can be modeled
as the expenditure of privacy budget by a client.
That is, clients treat the creation of a report
from private information
as a limited release of information.

Total privacy loss in this case is determined
by the combination of two factors:

* How the report is aggregated.

* How many reports are produced.

If aggregation includes the application of
an appropriate differential privacy mechanism
(that is, added noise;
see {{?DWORK=DOI.10.1561/0400000042}},
{{?DAP-DP=I-D.wang-ppm-differential-privacy}},
and {{Section 7.5 of DAP}}),
the client might rely on an understanding of that mechanism
to model privacy loss.
However, such privacy loss might be based on
an assumption that the client contributes just one report.
A complete model needs to consider
the contributions of multiple reports.

A privacy budgeting system
provides additional flexibility.
Privacy loss associated with any task
(or information release)
can be adjusted to control the amount of noise that is added.
A budget might be specified in terms of a metric
(like the epsilon parameter in (ε, δ)-differential privacy)
that is expended with each information release.

In one version of that model,
a client is responsible for the management of any privacy budget.
Each report represents a logical information release,
contingent on being sent for aggregation
by aggregators that are trusted to apply
an appropriate differential privacy mechanism
with the appropriate level of noise.


## Privacy Budget Upload Extension Format

The privacy_budget upload extension
(codepoint 0xTBD)
encodes the amount of privacy budget
that the client considers to be expended
as a result of producing a report.

The value of the codepoint is
an encoding of the number of milli-epsilons
of budget that are expended,
using as many bytes as needed to encode the value
in network byte order.
Each unit is a one-thousandth of an epsilon (ε) as used in
(ε, δ)-differential privacy.

{:aside}
> Note(1): Where the delta (δ) value is non-zero,
> and small epsilon increments can be expended,
> clients might also need to limit the number of reports
> to prevent the overall delta value from getting large.

{:aside}
> Note(2): A separate upload extension could be defined
> to change the scale of this value
> or switch to a different unit, as necessary.


## Privacy Budget Usage

An aggregator that is configured to apply
a differential privacy mechanism can operate in one of two modes:
either one where the privacy budget value is validated
and reports that contain a small value are rejected;
or, where the minimum privacy budget value is used
to determine the parameters for the differential privacy mechanism.

In the first mode,
aggregators each validate this parameter
as part of validating each report.
The value in the report is compared
with the value configured for the task.
A report that contains a value that is
lower than the value configured for the task
is the result of a client that expects that
the aggregators will add more noise
than the task configuration presently allows.
Aggregators MUST reject reports with a privacy budget value
that is smaller than their configured privacy budget.

Alternatively,
aggregators could adjust the parameters
of the differential privacy mechanism they use
to match the smallest privacy budget that was
included in reports.
For long-running tasks that produce multiple outputs over time,
it is only necessary to ensure that each output
contain noise that is based on the minimum budget expenditure
of the reports that are included in that aggregate.

This upload extension can be used
to protect reports that are conveyed from client
by untrusted entities,
especially where those entities might be able to choose any task,
as enabled by the late_binding upload extension ({{late-bind}}).
This parameter ensures that the entity cannot direct reports
to a task that has an inadequate differential privacy mechanism.



# Security Considerations

The security considerations relevant to each extension
is enumerated in the respective sections:
{{late-bind}},
{{requester}},
{{partition}},
and {{budget}}.

Use of DAP is subject to the security considerations
of DAP ({{Section 7 of DAP}})
and the VDAF that is in use ({{Section 9 of VDAF}}.


# IANA Considerations

Registrations for the defined upload extensions need to be made,
but this depends on the resolution of the TODO
in {{Section 8.2.2 of DAP}}.


--- back

# Acknowledgments
{:numbered="false"}

Roxana Geambesu noted that a binding to requester identity ({{requester}})
was an important component of a robust differential privacy system design.
