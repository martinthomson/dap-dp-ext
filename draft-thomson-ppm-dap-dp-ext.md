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
This includes bindings of measurements to specific options,
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
the same measurement cannot be aggregated more than once.
An honest participant will contribute a limited number of measurements
and can rely on at least one aggregator
preventing that measurement from being used multiple times.
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
The identity of the task is bound to each measurement
through the inclusion of the task ID in the call to the sharding function
of the VDAF ({{Section 5.1 of ?VDAF=I-D.irtf-cfrg-vdaf}}).

The late-binding extension (codepoint 0xTBD)
signals to aggregators that a measurement was not bound to a specific task
when it was created.

Late binding might be useful when measurements are collected by an intermediary,
where the client that generates the measurement is not aware of
how the measurement will ultimately be aggregated.
This allows the intermediary to defer the creation of a task
until it has determined the necessary parameters for the task.

When sharding and protecting measurements,
the task ID is replaced with a the fixed, 32-byte sequence of
b13e8440f1cdb4da51eed3967e0a2652d27f5005bc35f751daf188b4b746708b
(in hex).
(This is the output from SHA-256 {{?SHA2=RFC6234}}
when passed an ASCII-encoded {{?ASCII=RFC0020}} input of
'no task_id'.)

Enforcing anti-replay for a measurement
that is not bound to a specific task
is challenging.
An aggregator cannot constrain its search for duplicate measurements
to those that were submitted to the task.
This could greatly increase the cost of meeting anti-replay requirements.
The intent with this extension is that additional constraints,
such as one or more of the scoping extensions (see {{scoping}}),
will be used to make it more feasible
for an aggregator to comply with anti-replay requirements.


# Scoping Extensions {#scoping}

The DAP upload extensions in this section might be used to either
constrain the use of measurements
for tasks that are configured with matching values
or group measurements for the purposes of detecting duplicates.

Including additional scoping information can also
ensure that measurements do not get reused
outside of their intended scope.


## Requester (Website) Identity {#requester}

Measurements might be requested by
an entity that operates at lower trust level than
the entity that assembles the measurement.
The entity at the lower trust level
might not have access to the information necessary
to generate the measurement.

For example, an application could ask the operating system
to generate a measurement
using information that would normally be withheld from it.
Similarly, a website could ask a web browser to generate a measurement.
In either case,
the release of information for measurement
is conditional on it only being used
by a specific aggregation service
under terms that have been previously established
with the aggregators.
Binding the measurement to the identity of the requester
ensures that any use of the system can be accounted for
as coming from that requester.

The specific encoding used in this extension
will depend on the application.
However, the use of a globally-unique identifier,
such as an origin ({{?ORIGIN=RFC6454}})
or serialized site ({{SITE}}),
reduces the odds of name collisions.
A name collision might either allow two requesters
that share an aggregator
to share and reuse each others measurements
(or perhaps to marginally increase the odds
of having measurements spuriously detected duplicates).


## Thread


# Privacy Budget {#budget}


# Security Considerations

TODO Security


# IANA Considerations

Registrations for the defined upload extensions need to be made,
but this depends on the resolution of the TODO
in {{Section 8.2.2 of DAP}}.


--- back

# Acknowledgments
{:numbered="false"}

Roxana Geambesu noted that a binding to requester identity ({{requester}})
was an important component of a robust differential privacy system design.
