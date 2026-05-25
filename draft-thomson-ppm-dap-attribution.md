---
title: "Distributed Aggregation Protocol (DAP) Extensions for the Attribution API"
abbrev: "DAP Extensions for Attribution"
category: std

docname: draft-thomson-ppm-dap-attribution-latest
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
  github: "martinthomson/dap-attribution"
  latest: "https://martinthomson.github.io/dap-attribution/draft-thomson-ppm-dap-attribution.html"

author:
  -
    fullname: "Martin Thomson"
    organization: Mozilla
    email: "mt@lowentropy.net"

normative:

informative:
  ATTR:
    target: https://w3c.github.io/attribution
    title: "Attribution API"
    date: 2025-01-14
    author:
      - name: Andrew Paseltiner
      - name: Andy Leiserson
      - name: Benjamin Case
      - name: Benjamin Savage
      - name: Charlie Harrison
      - name: Martin Thomson
  ATTR-DP:
    target: https://arxiv.org/abs/2506.05290
    title: "Beyond Per-Querier Budgets: Rigorous and Resilient Global Privacy Enforcement for the W3C Attribution API"
    date: 2025-10-14
    author:
      - name: Pierre Tholoniat
      - name: Alison Caulfield
      - name: Giorgio Cavicchioli
      - name: Mark Chen
      - name: Nikos Goutzoulias
      - name: Benjamin Case
      - name: Asaf Cidon
      - name: Roxana Geambasu
      - name: Mathias Lécuyer
      - name: Martin Thomson
  CAP-URL:
    target: https://www.w3.org/TR/capability-urls/
    title: Good Practices for Capability URLs
    date: 2014-02-18
    author:
      - name: Jeni Tennison
  SHUFFLE:
    target: https://arxiv.org/abs/2001.03618
    title: "Encode, Shuffle, Analyze Privacy Revisited: Formalizations and Empirical Evaluation"
    author:
      - name: Úlfar Erlingsson
      - name: Vitaly Feldman
      - name: Ilya Mironov
      - name: Ananth Raghunathan
      - name: Shuang Song
      - name: Kunal Talwar
      - name: Abhradeep Thakurta
    date: 2020-01-10
  SITE:
    target: https://html.spec.whatwg.org/#site
    title: HTML - Living Standard
    date: 2021-01-26
    author:
      - org: WHATWG
  WEB-PRIV:
    target: https://w3ctag.github.io/privacy-principles/
    title: "Privacy Principles"
    date: 2025-11-24
    author:
      - name: Robin Berjon
      - name: Jeffrey Yasskin


--- abstract

This defines extensions to the DAP protocol
that support the Attribution API.
These extensions provide support for differentially-private aggregation
and the operating modes that the Attribution API depends on.


--- middle

# Introduction

The Attribution API {{ATTR}} is a web platform feature
that provides sites that use advertising
with the ability to measure the effectiveness of that advertising.
Measurements that the API produce are aggregated
using the Distributed Aggregation Protocol (DAP) {{!DAP=I-D.ietf-ppm-dap}}.

This document defines extensions to DAP
that support the use of DAP by the Attribution API.
These extensions fall into two categories:

* Support for the differential privacy model
  used in the Attribution API.
* Support for the operational modes
  that better fit the deployment model that the Attribution API uses.

These extensions are not narrowly defined
such that they can only be used by the Attribution API.
Other applications might be able to use them,
but any effort to make them fully generic
stops short of making the extensions more complex
than is required for Attribution.

For example, privacy budget extensions are defined
to use the epsilon definition
from (ε, 0)-differential privacy
or (ε, δ)-differential privacy with a fixed δ value.
This is simpler than a design
that might allow for multiple budget metrics.


## Differential Privacy in Attribution

The Attribution API provides information to websites
based on user activity on other websites,
which is ordinarily prohibited for privacy reasons {{WEB-PRIV}}.

To protect privacy,
the Attribution API uses differential privacy {{?DP=DOI.10.1561/0400000042}}
in a combination of the central and individual models {{ATTR-DP}}.
The privacy architecture of the Attribution API
allocates responsibility for managing sensitivity and privacy budgets
to browser instances (DAP Clients)
and for the addition of noise
to an aggregation service (DAP Aggregators, collectively).

To this end,
reports that are submitted for aggregation need to be bound
to the privacy budget that was expended at a Client
when the report was generated.
This ensures that Aggregators can apply noise
with sufficient amplitude
to maintain the intended differential privacy guarantee.

An extension that reports the amount of privacy budget
that was consumed in the generation of a report
is described in {{dp}}.


## Attribution Operating Mode Extensions

The Attribution API is expected to operate in a mode
that differs somewhat from the way that DAP is architected.
Several extensions are defined to support this operating mode.

In DAP, a task is a long-running context
that Clients continuously contribute to.
Reports are directly uploaded by Clients to the Leader
as they are generated.
The DAP batch mode determines how reports are grouped for aggregation.
A new collection job is initiated by the Collector
when an aggregate is needed,
though this might fail if the requirements for the task --
the batch mode and minimum batch size, primary --
are not met.

A simple representation of the DAP architecture
is illustrated in {{f-arch-dap}}.

~~~ aasvg
                            +--------+
                            | Helper |
                            +--------+
                                ^
                                ║ Validate &
+---------+                     ║ Aggregate
| +-------+-+    Reports        v
+-+ +-------+-+ ----------> +--------+  Collection  +-----------+
  +-+ Clients | ----------> | Leader | <----------- | Collector |
    +---------+ ----------> +--------+ -----------> +-----------+
                                          Result
~~~
{: #f-arch-dap title="Simplified DAP Architecture" artwork-svg-options="--spaces=2"}

In the Attribution API,
reports are delivered to the website that requests them.
The site is expected to gather a bundle of reports
and submit them for aggregation
when they have a sufficiently large set.

An important feature of this design is that the website
(as a DAP Collector)
chooses which reports to aggregate.
This gives the site an opportunity to review the circumstances
in which reports were generated
and filter reports according to their needs.

A secondary reason for Clients to deliver reports to the website,
rather than submit them directly to the Leader,
is that this removes a real-time dependency on the Leader.
A Leader can therefore be less available
than the sites that depend on its services.

A simple representation of the architecture used by the Attribution API
is illustrated in {{f-arch-attr}}.

~~~ aasvg
                                                            +--------+
                                                            | Helper |
                                                            +--------+
                                                                ^
+------------+                                                  ║ Validate &
| +----------+-+    Reports                                     ║ Aggregate
| | +----------+-+ ----------> +-----------+   Batched          v
+-+ |  Clients   | ----------> | Collector |   Reports      +--------+
  +-+ (Browsers) | ----------> | (Website) | =============> | Leader |
    +------------+ ----------> +-----------+ <------------- +--------+
                                               Result
~~~
{: #f-arch-attr title="Attribution API Architecture" artwork-svg-options="--spaces=2"}

To support this mode of operation,
a new batch mode for DAP is defined
called "collector-selected"; see {{batch-mode}}.
For other batch modes,
the Leader has sufficient information to select reports
for inclusion in a collection job.
In comparison,
this batch mode requires that reports carry an annotation
set by the Collector
at the time that each report is uploaded.


## Other Extensions

Three other extensions are defined in this document.

The minimum privacy budget collection job extension ({{min-dp}})
added to collection job initialization
places guardrails around what reports can be included
in a collection job.
Reports that lack a privacy budget extension
or those with a value below the indicated threshold
must be rejected by Aggregators.

For the Attribution API,
setting a minimum privacy budget
is roughly equivalent to capping the noise
that is added to an aggregate.
Without this extension,
noise could be determined from the privacy budget values
bound to each report.
This ensures that reports that do not match expectations
can be dropped efficiently,
rather than having Aggregators add unexpectedly large amounts of noise.

The collector identity task extension ({{collector-id}})
binds the identity of the Collector to tasks.
This extension fixes the entity can request collection of reports.

In the Attribution API,
the Collector is either a "conversion site",
or an "intermediary site"; see {{ATTR}}.
The conversion site is the top-level site where reports are generated.
An intermediary site is any entity that operates
independently from the top-level site,
and it includes the providers of secondary resources (such as images)
and framed content (that is, HTML iframes).

The budget source task extension ({{budget-source}})
binds the identity of the context that provides privacy budget
to tasks.
This extension ensures that reports
that draw from different privacy budgets
cannot be aggregated together
without the allowances necessary to maintain differential privacy
(most likely, the addition of more noise).

For the Attribution API,
each "conversion site" receives their own source of privacy budget.
Binding tasks to their identity
ensures that the privacy guarantees
associated with that privacy budget hold
when Aggregators are responsible for adding noise.
That is, reports that draw from different budgets
cannot be aggregated together with a single quantity of noise.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document relies on the definitions in DAP {{!DAP}}
for protocol roles and functions.
Some reference is made to the terms and concepts in the Attribution API {{ATTR}},
though familiarity with these should not be necessary
to understand how the extensions interact with DAP.


# Differential Privacy Budget Report Extension {#dp}

The privacy budget report extension (see {{Section 4.4.3 of DAP}})
codepoint 0xTBD,
encodes the amount of privacy budget
that might have been expended by a Client
as a result of producing a report.

The value of the codepoint is
an integer encoding of the number of micro-epsilons
of budget that are expended.
That is, each unit is a one-millionth of an epsilon (ε)
as used in (ε, δ)-differential privacy.

The micro-epsilon value is encoded as a 32-bit integer
in network byte order.
This permits expenditure of up to ε=4294.967295 to be encoded.


## Privacy Budgeting

A privacy budget ensures that the total information release
can be bounded
while providing more flexibility to the recipients
of the noisy results.
Recipients are able to adjust how budget is used
to control how noise is distributed
across multiple information releases.

The amount of noise added to aggregates
is based on the expended budget.
In general,
spending more privacy budget means that less noise is needed
to maintain the same level of privacy;
conversely, spending less budget means more noise.

A budget might be specified in terms of a metric
(like the epsilon parameter in (ε, δ)-differential privacy)
that is expended with each information release.
As noted, this extension uses the ε metric.

For example, for an overall budget of ε=10
might be split four ways: (0.5, 1.5, 2, 6).
Noise might then be added,
drawing from a distribution
with a width inversely proportional to the budget spent;
that is, a distribution with a standard deviation proportional to
2, 2/3, 1/2, and 1/6 respectively.


## Applicability to Differential Privacy Models

The extensions in this document ({{dp}} and {{min-dp}})
only support ε-differential privacy
or (ε, δ)-differential privacy with a fixed delta (δ).
Systems that use other budgeting methods
require the use of a different extension.

A delta (δ) parameter is not directly bound to reports.
This parameter is rarely used in privacy budgeting.
Instead, a maximum value for δ might be fixed
as part of the configuration in a specific deployment.
Setting a value for δ is necessary
when selecting a differential privacy mechanism.
In setting a value for δ,
a deployment needs to consider the total report volume
and the total number of tasks that each client might contribute to.

{:aside}
> Note: Where the delta (δ) value is non-zero,
> and a client might generate many reports,
> clients might also need to limit the number of reports
> to prevent the overall delta value from growing large.


## Use in the Attribution API

When used in the Attribution API {{ATTR}},
the privacy budget report extension does not always encode the exact amount
of privacy budget that was expended.
The individual DP model used in Attribution (see {{ATTR-DP}}),
allows a Client to expend less of its budget than this value
for several reasons.

This introduces some complexity,
because any reduction in the budget spent
is based on private information.
Consequently, any reduction in budget expenditure needs to be kept secret.
In that setting,
the extension reports the requested expenditure,
which is the maximum value that might have been spent.

This extension gives users of the Attribution API
the ability to manage privacy budget expenditure
on a per-report basis.

By binding the amount of budget spent to each report,
the Client can transfer responsibility
for applying noise to Aggregators.
The addition of noise in a single place
can ensure a better trade-off between
the amount of added noise
and privacy parameters.

The Attribution API differs from some other uses of DAP.
which instead involve Clients adding noise to reports
at the time they are generated.
To maintain the usefulness of aggregates,
the amount of noise added by each Client is kept low.
To maintain strong privacy,
Clients partly rely on DAP mixing their contributions
with the contributions of other Clients,
following the shuffle DP approach {{SHUFFLE}}.
Such uses depend on attaining a certain minimum batch size
in order to meet their differential privacy targets.

Applying noise during aggregation
reduces the importance of the minimum batch size
parameter in task configuration.
However, it adds to the work
that Clients need to trust Aggregators to perform.

To maintain consistency with the DAP threat model for privacy
(see {{Section 8 of !DAP}}),
this can mean either performing noise addition in an MPC protocol
or having Aggregators each independently apply the requisite amount of noise.

A number of MPC protocols for adding noise exist,
but these are often not efficient in the two-party setting
relative to the VDAF protocols typically used.
On the other hand,
the redundant addition of noise by Aggregators
results in more noise when both Aggregators are honest.
Even so,
adding two amounts of noise
often results in less overall noise
than other approaches.


# Collector-Selected Batch Mode {#batch-mode}

The collector-selected batch mode
(codepoint 0xTBD)
give the Collector control over which reports are included in collection job.

In this batch mode
the Leader is not able to determine the associated batch for a report
based on the contents of each report alone.
This differs from the existing leader-selected ({{Section 5.2 of ?DAP}})
or time interval ({{Section 5.1 of ?DAP}}) batch modes.

To that end,
uploads of reports use a different URL template
from the usual location for report uploads;
see {{upload}}.

This arrangement prevents the Leader from accepting reports
until a collection job is created.
A Leader MAY accept hold requests to upload reports
for a short period prior to the creation of a collection job.
Accepting reports would be on the expectation
that a request to create the indicated collection job
is imminent or still being processed,
so it could reduce latency.

However, the Leader MUST limit the reports it accepts
prior to accepting the corresponding collection job.
Without a limit,
the Leader gives malicious actors
the unrestricted capability to exhaust its resources.

A Leader that does not accept reports MUST reject the request,
and can respond with an indication to try later.
This response could use a 404 status code
and the `Retry-After` response field;
see {{Sections 15.5.5 and 10.2.3 of ?HTTP=RFC9110}}.

{:aside}
> Where a collection job already exists,
> an high entropy and unpredictable collection job ID in the URL
> could make it unnecessary to require authentication of upload requests
> for this batch mode; see {{CAP-URL}}.
> This is not the case if reports are accepted
> without confirming the existence of the identified collection job
> or the collection job ID is more easily guessed.

A Leader MUST reject attempts
to upload reports to the regular report upload resource
(as defined in {{Section 4.5.2 of !DAP}})
when the collector-selected batch mode is configured for a task.


## Report Upload URL Template {#upload}

Reports in the collector-selected batch mode are uploaded
to a URL that follows the template:

~~~
{leader}/tasks/{task-id}/reports/{collection-job-id}
~~~

The inclusion of the collection job ID
(see {{Section 4.7 of !DAP}})
differs from other batch modes
where the Collector does not need to provide
any additional information to a Leader.


## Batch Mode Parameters {#mode-params}

This batch mode uses the same parameters
as the leader-selected batch mode ({{Section 5.2 of !DAP}}).
That is, the payload Query.config is empty.
Both the PartialBatchSelector.config
and the BatchSelector.config contains a batch ID
that is assigned by the Leader.


# Minimum Privacy Budget Collection Job Extension {#min-dp}

The minimum privacy budget collection job extension (see {{Section 4.6.2 of DAP}}),
codepoint 0xTBD,
allows a Collector to inform Aggregators
of a minimum value for the privacy budget report extension
(see {{dp}})
that it expects to be included in the collection job.

The format of this extension
is a 32-bit network-endian encoding of an integer
in units of micro-epsilon.
This is identical to the format of the privacy budget report extension ({{dp}}).

This extension is defined primarily as a safeguard.
In the absence of this extension,
Aggregators could determine the amount
of privacy budget that was expended by every report
and generate noise based on the minimum value across all reports.

That approach is potentially error prone.
If a Collector accidentally includes a report with a much lower budget,
the Aggregate it receives would have more noise added than expected.
The collection job extension effectively sets a cap
on the noise that might be added.

Aggregators can use this extension
in one of two ways:

* As an optimization and safeguard.
  The value in the collection job extension
  directly determines the magnitude of the noise that is added
  to the aggregate.
* As a safeguard alone.
  The value is only used to filter reports
  and the minimum value of the privacy budget report extension
  across all accepted reports determines the magnitude of added noise.

Either approach maintains differential privacy guarantees.
The latter can result in adding less noise
in the case that the Collector provides a low value,
but is more complex to implement.
There is also no way provided to indicate to the Collector
that less noise was added than they might have planned.


# Collector Identity Task Extension {#collector-id}

The collector identity task extension (see {{Section 4.2.2 of DAP}}),
codepoint 0xTBD,
binds the task --
and all reports submitted to that task --
to a single Collector.

This extension does not specify how to encode the identity of the Collector.
Different uses of DAP can choose an encoding
that best suits the needs of the application.

The Attribution API has its own understanding
of how to encode the identity of the Collector.
The value is a UTF-8-encoded string
of the registrable domain
from the intermediary site tuple (if there is an intermediary site)
or the conversion site tuple (where there is no intermediary site).


## Collector Identity and HPKE Configuration

Regardless of how the Collector is identified,
if an identity is included,
the Leader and Helper MUST have a process
for validating the HPKE configuration they use
to encrypt aggregate shares for the Collector.

That process MUST provide confirmation that the identified entity
authorizes the HPKE configuration.
This does not depend on active proof of possession
for the corresponding private key {{?SIGMA=DOI.10.1007/978-3-540-45146-4_24}}
but rather an affirmation that the public key
is approved by the identified entity.

One option for Collector identification is to use a URL,
following the pattern used for the Leader and Helper;
see {{Section 4.2 of DAP}}.
If the URL uses an authenticated protocol,
such as HTTP with the "https" scheme {{?RFC9110}},
retrieving an HPKE configuration from that URL
(or similarly authenticated resources
that are referenced from the response)
provides the necessary authorization for the included key.

The Attribution API does not define a process
for authorizing a Collector HPKE configuration
based on the encoded Collector identity,
leaving this to implementations.


# Privacy Budget Source Task Extension {#budget-source}

The privacy budget source task extension (see {{Section 4.2.2 of DAP}}),
codepoint 0xTBD,
binds a task --
and all reports submitted to that task --
to a single source of privacy budget.

This extension does not specify how to encode the identity of this entity.
Different uses of DAP can choose an encoding
that best suits the needs of the differentially private usage.

Any usage of this design needs to ensure that different sources of privacy budget
have different unique values that are encoded into this extension.
This binding ensures that reports that are attributed to different budgets
cannot be aggregated together,
which would violate differential privacy goals.

{:aside}
> In theory, the need for this sort of task diversification
> could be fulfilled by the `task_info` parameter
> in the task configuration encoding.
> However, a separate task extension ensures that `task_info`
> remains available for other uses.

The Attribution API has its own understanding
of how to encode the identity of the budget source.
The value is a UTF-8-encoded string
of the registrable domain
from the conversion site tuple.


# Security Considerations

Security factors specific to each extension
are covered in the respective sections.

Use of DAP is subject to the security considerations
of DAP ({{Section 8 of DAP}})
and the VDAF that is in use ({{Section 9 of !VDAF=I-D.irtf-cfrg-vdaf}}.


# IANA Considerations


This document registers a new batch mode
in the "DAP Batch Mode Identifiers" registry
established in {{Section 9.2.1 of DAP}}.

| Value  | Name               | Reference      |
|:-------|:-------------------|:---------------|
| TBD    | collector_selected | {{batch-mode}} |
{: #t-dap-bm title="DAP Match Mode"}

This document registers task extensions
in the "DAP Task Extension Identifiers" registry
established in {{Section 9.2.2 of DAP}}.

New task extensions are tabulated
in {{t-dap-task-ext}}.

| Value  | Name               | Reference          |
|:-------|:-------------------|:-------------------|
| TBD    | collector_identity | {{collector-id}}   |
| TBD    | budget_source      | {{budget-source}}  |
{: #t-dap-task-ext title="Task Configuration Extensions"}

This document registers a DAP report extension
in the "DAP Report Extension Identifiers" registry
established in {{Section 9.2.3 of DAP}}.

The new report extension registration is tabulated
in {{t-dap-report-ext}}.

| Value  | Name               | Reference     |
|:-------|:-------------------|:--------------|
| TBD    | privacy_budget     | {{dp}}        |
{: #t-dap-report-ext title="DAP Extensions"}

This document registers a collection job extension
in the "DAP Collection Job Extension Identifiers" registry
established in {{Section 9.2.4 of DAP}}.

The new collection job extension is tabulated
in {{t-dap-collect-ext}}.

| Value  | Name               | Reference          |
|:-------|:-------------------|:-------------------|
| TBD    | min_dp_budget      | {{min-dp}}         |
{: #t-dap-collect-ext title="Collection Job Extensions"}


--- back

# Acknowledgments
{:numbered="false"}

{{{Roxana Geambesu}}} noted that a binding to site identity ({{collector-id}})
was an important component of a robust differential privacy system design
for the Attribution API.
{{{David Cook}}} provided useful feedback about the design and document.
{{{Chris Patton}}} provided helpful input on how to integrate with the DAP architecture.
