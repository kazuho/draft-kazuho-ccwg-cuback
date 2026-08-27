---
title: "CUBACK: CUBIC Driven by the ACK Clock"
abbrev: "cuback"
category: std
docname: draft-kazuho-ccwg-cuback-latest
workgroup: "Congestion Control Working Group"
ipr: trust200902
keyword: internet-draft
pi: [roc, sortrefs, symrefs]
author:
 -
    fullname:
      :: 奥 一穂
      ascii: Kazuho Oku
    org: Fastly
    email: kazuhooku@gmail.com

normative:

informative:

...

--- abstract

This document specifies Cuback, an ACK-driven reformulation of CUBIC congestion
control that simplifies implementation by replacing CUBIC's mutable time- and
ACK-driven state with pure functions over immutable per-epoch parameters.
Congestion-window growth then uses the same ACK-driven mechanism as Reno. Test
vectors are provided for validation.


--- middle

# Introduction {#intro}

CUBIC {{!RFC9438}} is a widely deployed congestion-control algorithm that
improves scalability over Reno by defining congestion-window growth as a cubic
function of elapsed time. Its implementation, however, is more complex than the
growth function itself might suggest.

During congestion avoidance, a CUBIC sender maintains two independently evolving
window estimates: the time-driven cubic window and the ACK-driven Reno-friendly
window. The sender must update both estimates and select the larger of the two.
In addition, because the cubic window advances with wall-clock time, the sender
needs mutable epoch state and a state machine that pauses the cubic clock when
the sender ceases to be congestion-control limited and resumes it when
transmission becomes limited by the congestion window again. Correctly
maintaining these states complicates implementations and their interaction with
application-limited periods and other congestion-control transitions.

This document specifies Cuback, an ACK-driven reformulation of CUBIC. Cuback
expresses both the cubic and Reno-friendly growth functions in terms of the
amount of data that must be acknowledged for the congestion window to reach a
given value. The parameters defining those functions are established at the
beginning of a congestion-avoidance epoch and remain immutable for the duration
of that epoch. Congestion-window growth can therefore be calculated from a pure
function of the current congestion window and the per-epoch parameters, without
maintaining separately evolving cubic and Reno-friendly window estimates.

This formulation also eliminates the need to pause and resume a
wall-clock-driven CUBIC epoch. When acknowledgments stop arriving, advancement
along the Cuback curves stops naturally. The remaining runtime state is
essentially the same as in Reno: the sender tracks how many additional bytes
need to be acknowledged before increasing the congestion window by one MSS.

Cuback therefore retains the characteristic growth behavior of CUBIC while
reducing congestion avoidance to the same ACK-driven window-increase mechanism
used by Reno. In both Reno and Cuback, acknowledged bytes are accumulated until
enough have been received to increase the congestion window by one MSS. The
difference lies only in how the required number of acknowledged bytes is
calculated: Reno derives it directly from the current congestion window, whereas
Cuback obtains it by evaluating the pure functions defined in this document.

This property also makes the algorithm straightforward to test: the
congestion-avoidance calculations can be validated directly from their inputs
and outputs, without exercising a sequence of state-machine transitions. This
document includes test vectors for that purpose.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
