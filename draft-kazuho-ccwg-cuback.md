---
title: "CUBACK: CUBIC Driven by the ACK Clock"
abbrev: "cuback"
category: std
docname: draft-kazuho-ccwg-cuback-latest
workgroup: "Congestion Control Working Group"
ipr: trust200902
keyword: internet-draft
pi: [roc, sortrefs, symrefs]
stand_alone: yes
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

CUBIC {{!CUBIC=RFC9438}} is a widely deployed congestion-control algorithm that
improves scalability over Reno {{?RENO=RFC5681}} by defining congestion-window
growth as a cubic function of elapsed time. Its implementation, however, is more complex than the
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

Cuback alters only the congestion avoidance stage of CUBIC. All other behavior
specified in {{CUBIC}} applies unchanged.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the notation of {{CUBIC}}, in which window sizes are
expressed in segments. An implementation that counts bytes instead scales each
window value by SMSS, divides the sums appearing in {{ca}} by SMSS, and
expresses the lower bound on D(w) as 2 * SMSS.


# Cuback Congestion Avoidance {#ca}

During congestion avoidance, a Reno sender accumulates the data acknowledged and
increases cwnd by one segment each time a threshold D(cwnd) is reached:

~~~
on entering congestion avoidance:
  pending = 0

on receiving an acknowledgement, while cwnd >= ssthresh:
  pending = pending + segments_acked
  while pending >= D(cwnd):
    pending = pending - D(cwnd)
    cwnd = cwnd + 1
~~~

In Reno, that threshold is the congestion window itself:

~~~
D_reno(cwnd) = cwnd
~~~

Cuback retains the mechanism above unchanged and replaces only the threshold,
where A(w) is the amount of data, counted from the beginning of the current
congestion avoidance stage, that has to be acknowledged for the congestion
window to reach w:

~~~
D(w)       = max(2, A(w + 1) - A(w))

A(w)       = min(A_cubic(w), A_reno(w))

A_cubic(w) = bandwidth * (K(W_max, cwnd_epoch) + cbrt((w - W_max) / C))

A_reno(w)  = (w - cwnd_epoch) * (w + cwnd_epoch - 1)
               / (2 * alpha_cubic)                            w <= W_max

           = (W_max - cwnd_epoch) * (W_max + cwnd_epoch - 1)
               / (2 * alpha_cubic)
             + (w - W_max) * (w + W_max - 1) / 2              w >  W_max

K(W_max, cwnd_epoch) = cbrt((W_max - cwnd_epoch) / C)
~~~

W_max, bandwidth, and cwnd_epoch are fixed when the congestion avoidance stage
begins and do not change until the next congestion event:

* W_max is as defined in {{Section 4.1.2 of CUBIC}}. It is set to cwnd_prior,
  the congestion window before the reduction at the congestion event that began
  the stage.

* bandwidth is an estimate of the rate at which the bottleneck delivers data, in
  segments per second. It is set at the congestion event to cwnd_prior divided
  by the smoothed round-trip time.

* cwnd_epoch is the congestion window at the beginning of the congestion
  avoidance stage, as defined in {{Section 4.1.2 of CUBIC}}.

alpha_cubic, beta_cubic, and C are the constants defined in {{CUBIC}}, with
recommended values:

~~~
beta_cubic  = 0.7
alpha_cubic = 0.529  # 3 * (1 - beta_cubic) / (1 + beta_cubic)
C           = 0.4
~~~

A sender that derives ssthresh from flight_size rather than from cwnd
({{Section 4.6 of CUBIC}}) MUST use flight_size in place of cwnd_prior
throughout this section. W_max, cwnd_epoch, and bandwidth MUST be derived from
the same quantity.

When congestion avoidance is entered without a congestion event, such as on
exiting slow start, the parameters are established at that point instead. cwnd
is not reduced, so W_max equals cwnd_epoch and K is zero, and A(w) describes
growth along the convex region alone.

The expressions above are derived in {{derivation}}.

## Fast Convergence {#fast-convergence}

Fast convergence is applied as described in {{Section 4.7 of CUBIC}}: on a
congestion event, if cwnd is below W_max, W_max is set to
cwnd * (1 + beta_cubic) / 2 before the window is reduced. A_reno in {{ca}} then
uses
cwnd_prior, which is cwnd_epoch / beta_cubic, in place of W_max, {{Section 4.3
of CUBIC}} keying the switch to alpha_cubic = 1 to cwnd_prior rather than to
W_max.

## Spurious Congestion Events {#spurious}

When a congestion event is determined to have been spurious ({{Section 4.9 of
CUBIC}}), a sender that reverts cwnd and ssthresh MUST also restore W_max,
bandwidth, and cwnd_epoch to the values they held before the event.

## Application-Limited Senders {#app-limited}

A sender that can detect application-limited periods SHOULD refrain from
increasing cwnd during them.

Unlike in {{CUBIC}}, this is not mandatory, Cuback being ack-clocked. See
{{app-limited-growth}}.

# Relationship to RFC 9438 {#relationship}

## Preserved Behavior {#preserved}

W_max, cwnd_prior, cwnd_epoch, K, alpha_cubic, beta_cubic, and C are those of
{{CUBIC}} and are established by the mechanisms of that document. Relative to
Section 4.1.2 of that document, Cuback adds bandwidth and removes t_current,
t_epoch, W_cubic(t), target, and W_est; apart from W_max and bandwidth, a Cuback
sender retains only pending, exactly as a Reno sender does. A_cubic is the
inverse of W_cubic(t), so the cubic trajectory is unchanged. The limit of
one half of the congestion window per round trip is preserved as the lower bound
on D(w). Where A_reno is the smaller of the two curves, D(w) is
cwnd / alpha_cubic, the Reno-friendly increase of {{Section 4.3 of CUBIC}}.

{{Section 4.4 of CUBIC}} and {{Section 4.5 of CUBIC}} approximate W_cubic(t) by
advancing cwnd
toward the target W_cubic(t + RTT). That look-ahead exists because a per-
acknowledgement additive rule can only approach the curve; A_cubic inverts it
exactly, and no look-ahead is required.

## The Reno-Friendly Estimate {#reno-estimate}

{{Section 4.3 of CUBIC}} advances W_est by
alpha_cubic * segments_acked / cwnd, where cwnd is the congestion window in
effect. While the cubic curve governs, that window is larger than W_est, so the
growth of W_est depends on the trajectory of the other curve. This coupling
cannot be expressed as a function of W_est alone, and A_reno instead inverts the
standalone recurrence, in which the divisor is the Reno-friendly window itself.

Where A_reno is the smaller of the two curves, cwnd is the Reno-friendly window
and the two formulations coincide exactly: both increase the window by one
segment for every cwnd / alpha_cubic segments acknowledged. They differ only in
where that region is entered. Because the standalone recurrence uses the smaller
divisor, it advances faster while the cubic curve governs, and a Cuback sender
enters the Reno-friendly region somewhat earlier than a sender following
{{CUBIC}}.

## Sensitivity to the Bandwidth Estimate {#sensitivity}

bandwidth scales A_cubic linearly, and so an error in it is an error in the rate
at which the cubic curve is traversed. Underestimating the delivery rate
traverses the curve more quickly than elapsed time would, and overestimating it
more slowly. A sender following {{CUBIC}} reads a clock and is not subject to
this error. This is the cost of the reformulation.

The estimate is derived from the sender's own congestion window and smoothed
round-trip time, so a receiver cannot influence it other than by inflating the
measured round-trip time, which lowers bandwidth and accelerates the traversal.
That effect is partly self-cancelling, as the round trip over which the
acknowledgements arrive lengthens correspondingly.

## Growth While Application Limited {#app-limited-growth}

{{Section 5.8 of CUBIC}} requires that the cubic curve not advance while the
sender is application limited, which entails suspending and resuming the stage.
In Cuback, A(w) advances only as data is acknowledged, so a sender that is not
filling its congestion window advances proportionally more slowly with no
additional mechanism.

This is not equivalent to suspending growth. An application-limited Cuback
sender continues to advance, at a rate bounded by the data it does have in
flight. The recommendation of {{app-limited}} therefore stands: Cuback bounds
the
consequence of imprecise detection rather than removing it.

The same distinction bears on the quantity from which the epoch is derived.
{{Section 4.6 of CUBIC}} sets ssthresh from flight_size while W_max comes from
cwnd (Section 4.1.2), because cwnd may have grown while the sender was not
filling it; {{Section 3.1 of RENO}} makes the same choice, warning that
cwnd "may incidentally increase well beyond rwnd". That growth is a property of
the clock. A CUBIC sender advances W_cubic(t) with elapsed time, so an epoch
that is not correctly suspended inflates cwnd while nothing is being delivered.
Cuback advances only as data is acknowledged, so the divergence between cwnd and
flight_size is bounded by the data the sender did place in flight. Cuback is
therefore in the position of Reno rather than of CUBIC, and the choice between
the two quantities is the one already made differently by {{RENO}}, which
uses flight_size, and by {{?RFC9002}}, which reduces cwnd directly.

This document does not choose between them; it requires only that W_max,
cwnd_epoch, and bandwidth be derived from the same one ({{ca}}). The cost of
mixing them falls on the early part of the epoch. A W_max taken from a window
the flow was not filling places the plateau above the one it sustained, and
because the cubic curve rises most steeply farthest from its inflection point,
the steps that become cheapest are those just after the congestion event -- when
the flows it competes with are themselves recovering. Deriving bandwidth from
that same window scales A_cubic and partly offsets the effect, which is why the
requirement here is consistency rather than a particular choice.

## Reverting an Epoch {#reverting}

Restoring the parameters ({{spurious}}) is all that reverting a congestion event
requires, because they are the sole inputs to A(w) besides the congestion window
itself. No trajectory has to be reconstructed.


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Derivation of the Increase Function {#derivation}

The lower bound of two segments in D(w) limits the increase to one half of the
congestion window per round trip: cwnd segments are acknowledged per round trip,
and each one-segment increase consumes at least two of them. It is the
counterpart of the constraint target <= 1.5 * cwnd in {{Section 4.4 of CUBIC}}
and {{Section 4.5 of CUBIC}}. No lower bound on the increase is needed, as A(w)
is monotonically
increasing and D(w) is therefore always positive.

CUBIC sets its congestion window to the larger of two window growth functions.
Both are monotonically increasing, so their maximum is monotonically increasing
as well, and the inverse of that maximum is the minimum of the two inverses.
A(w) is therefore obtained by inverting each function separately and taking the
smaller of the two results.

A_cubic is the inverse of W_cubic(t) of {{Section 4.2 of CUBIC}}, multiplied
by bandwidth to express elapsed time as acknowledged data. That the curve is
zero at w = cwnd_epoch follows from the definition of K.

Sampling bandwidth as cwnd_prior / RTT is the rate at which the flow was
delivering data over the round trip preceding the congestion event. It carries
the identity bandwidth * RTT = cwnd_prior: while the congestion window is at
cwnd_prior, one round trip of acknowledgements advances A(w) by exactly one
round trip along the underlying cubic curve.

A_reno is the inverse of the Reno-friendly increase of {{Section 4.3 of
CUBIC}}: one segment for every cwnd / alpha_cubic segments acknowledged,
with alpha_cubic replaced by one once the window reaches W_max. Summing those
thresholds over the segment-spaced windows from cwnd_epoch to w yields the
expression in {{ca}}.

Differencing A_reno recovers exactly the threshold from which it was built:

~~~
A_reno(w + 1) - A_reno(w) = w / alpha_cubic
~~~

The Reno-friendly region is therefore not a separate mechanism in Cuback. It is
the value that D(w) takes wherever A_reno is the smaller of the two curves. The
closed form exists only so that the two curves can be compared.

Differencing A_cubic eliminates K, so an implementation need not evaluate it
when the cubic curve is the smaller of the two:

~~~
A_cubic(w + 1) - A_cubic(w)
  = bandwidth * (cbrt((w + 1 - W_max) / C) - cbrt((w - W_max) / C))
~~~


# Acknowledgments
{:numbered="false"}

TODO acknowledge.
