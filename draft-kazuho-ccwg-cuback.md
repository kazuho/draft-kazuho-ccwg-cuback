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
congestion event, if cwnd is below W_max, W_max is reduced before cwnd is:

~~~
W_max = cwnd * (1 + beta_cubic) / 2
~~~

{{Section 4.3 of CUBIC}} keys the switch to alpha_cubic = 1 to cwnd_prior rather
than to W_max, so A_reno in {{ca}} uses cwnd_prior in its place; a sender that
does not retain it can recover it as cwnd_epoch / beta_cubic.

## Spurious Congestion Events {#spurious}

When a congestion event is determined to have been spurious ({{Section 4.9 of
CUBIC}}), a sender that reverts cwnd and ssthresh MUST also restore W_max,
bandwidth, and cwnd_epoch to the values they held before the event.

## Application-Limited Senders {#app-limited}

A sender that can detect application-limited periods SHOULD refrain from
increasing cwnd during them.

Unlike in {{CUBIC}}, this is not mandatory, Cuback being ack-clocked. See
{{app-limited-sensitivity}}.

# Relationship to RFC 9438 {#relationship}

## Sensitivity to the Bandwidth Estimate {#sensitivity}

bandwidth scales A_cubic linearly, so an error in it is an error in the rate at
which the cubic curve is traversed: an underestimate traverses the curve more
quickly than elapsed time would, and an overestimate more slowly. Cuback is
therefore sensitive to the smoothed round-trip time from which bandwidth is
derived, which has to describe the path as it was while cwnd was at its peak,
with the bottleneck queue at its deepest. A round-trip time carried over from a period
when the queue was shallower is too short and overstates bandwidth, leading to
growth more conservative than the cubic curve prescribes.

The sensitivity is confined to windows large enough for the cubic curve to
govern, A_reno carrying no bandwidth term. Windows that large deliver many
acknowledgements per round trip, and a QUIC sender takes a round-trip sample
from every acknowledgement that advances the largest acknowledged packet number
{{?RFC9002}}, so the smoothed round-trip time can be assumed to follow the queue
closely by the time the estimate matters.

## Convergence under the ACK Clock {#convergence}

Deriving the clock from acknowledgements reinforces the convergence that AIMD
provides, because bandwidth records the rate the flow was achieving before it
reduced. A flow holding a large share gives up the most at a congestion
event, so the rate it goes on to achieve falls below the value it recorded and
its clock runs slow. A flow arriving with a small share records a correspondingly
small rate and then achieves more than it recorded, so its clock runs fast. The
congestion window advances fastest for the flow gaining share and slowest for
the one giving it up.

## Sensitivity to Application-Limited State {#app-limited-sensitivity}

{{Section 4.2 of CUBIC}} requires that elapsed time exclude periods during which
cwnd was not updated because the sender was application limited. A CUBIC sender
therefore has to classify each moment as one state or the other, and the
classification can be wrong in either direction.

Treating an application-limited sender as congestion-window limited leaves the
clock running. W_cubic(t) advances while the flow is not filling cwnd, and cwnd
grows on evidence the path never supplied; {{Section 5.8 of CUBIC}} names the
consequence, that W_cubic(t) "might be very high after restarting from these
periods". Treating a congestion-window-limited sender as application limited
stops the clock instead, abandoning the epoch and re-anchoring it when sending
resumes, so the flow forfeits its progress along the curve and grows too slowly.

Neither state is directly observable: a sender that has filled its output batch
is sending, but that does not establish that it filled its congestion window,
and a paced sender waiting for its next send opportunity has filled the window
but is not sending at that instant. Tracking the state precisely enough for
CUBIC has needed a specification of its own, now written down in
{{?I-D.ietf-ccwg-ratelimited-increase}}, which updates {{CUBIC}}.

Cuback has no clock to stop or start. When the sender stops, the advance of A(w)
stops with it: no acknowledgements arrive, and nothing accrues to be caught up
when transmission resumes. The case in which a misclassifying CUBIC sender is
most exposed therefore does not arise.

A sender that keeps sending but holds less than cwnd in flight is mitigated
rather than protected. Growth is governed by the amount of data acknowledged, as
in Reno, so the window advances in proportion to what the flow placed in flight
rather than in proportion to elapsed time.

## The Reno-Friendly Estimate {#reno-estimate}

{{Section 4.3 of CUBIC}} advances W_est on each acknowledgement:

~~~
W_est = W_est + alpha_cubic * segments_acked / cwnd
~~~

where cwnd is the congestion window in effect. While the cubic curve governs, that window is larger than W_est, so the
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

## Smoothing the Cubic Growth {#smoothing}

{{Section 4.4 of CUBIC}} and {{Section 4.5 of CUBIC}} advance cwnd toward a
target one round trip ahead, W_cubic(t + RTT), rather than assigning W_cubic(t) to cwnd directly.
Because the CUBIC curve is driven by elapsed time, the growth owed at an
acknowledgement depends on how long has passed since the previous one, and
assigning the curve directly would turn a gap in acknowledgements into a step in
cwnd, which is a burst. The one-round-trip target, together with the
per-acknowledgement increase of (target - cwnd) / cwnd, spreads one round trip
of the curve across one round trip of acknowledgements, making the increase
proportional to the data acknowledged however the acknowledgements are
distributed in time.

Cuback needs no such smoothing. D(w) is a function of acknowledged data alone, so
a gap in acknowledgements accrues no growth to be caught up, and the increase is
proportional to the ack stream by construction.

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
an identity:

~~~
bandwidth * RTT = cwnd_prior
~~~

While the congestion window is at cwnd_prior, one round trip of acknowledgements
advances A(w) by exactly one round trip along the underlying cubic curve.

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
