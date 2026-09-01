# Telemetry Enhanced Additive Increase Multiplicative Decrease

<p align="right">
Pete Heist<br/>
Rodney Grimes<br/>
September 1, 2026
</p>

Before researching [CSTA](../csta/csta.md), we experimented with network
telemetry using [scim](https://github.com/heistp/scim) to enhance conventional
AIMD congestion control.  The results suggested that while AIMD is highly
robust, there are limits to scalability (in terms of both capacity and RTT),
that are hard to overcome without negatively impacting some other aspect of
performance.

In [telemetry.go](https://github.com/heistp/scim/blob/main/telemetry.go), there
are two experimental CCAs we worked with, TCP Stuttgart and TCP Liberec.

## TCP Stuttgart

TCP Stuttgart works by using the queue sojourn time from telemetry to
continuously remove this flow's contribution to the standing queue using a
technique we refer to as "cwnd targeting".  At any given time where a positive
sojourn is observed, cwnd can be set as follows, where sRTT is the smoothed RTT:

$cwnd = cwnd_\text{1-RTT-ago} - cwnd_\text{1-RTT-ago} * sojourn / sRTT$

The quantity $sojourn / sRTT$ can further be scaled by the current flow's rate
vs the bottleneck's rate, as measured by a total bytes sent counter in the
telemetry vs bytes sent by the sender.

To handle short term bursts, we save cwnd whenever making an adjustment due to
sojourn, and restore the saved cwnd if sojourn returns to 0 within an RTT.

We found that this strategy works, but with at least two drawbacks:
- There is a lot of cwnd noise, which leads to continuous variability in the
  queue length (jitter).
- High RTT flows got more capacity than low RTT flows (the opposite of
  conventional AIMD mechanics, which converge to equal cwnd), because the burst
  filter duration is tied to the RTT.

We experimented with a fixed duration burst filter and that led to flow fairness
within some limits, but then the question is, what duration is appropriate?
It's not an easy question to answer, as it depends on RTT, flow count and other
factors.  We sought another approach.

## TCP Liberec

With TCP Liberec, we switched to operating on the queue length at dequeue time,
instead of sojourn.  We tracked the minimum dequeue length within the last RTT
as the standing queue.  Once per RTT, if the minimum dequeue length is 0, we
simply add MSS to cwnd.  If the minimum dequeue length is greater than 0, we
adjust cwnd as follows:

* *fsq*: flow standing queue
* *minDQLen*: minimum dequeue length in the last RTT (in bytes), per telemetry
* *flowSent*: flow sent bytes in last RTT
* *bottleneckSent*: bottleneck sent bytes in last RTT, per telemetry

$fsq = minDQLen * flowSent / bottleneckSent$

$cwnd = cwnd_\text{2-RTTs-ago} - fsq/2 + MSS/2$

The above was found through intuition and experiment, and was partly inspired by
DCTCP.  It was found to be stable, and led to flow fairness independent of RTT
under many conditions (with some corner cases with degenerate RTT differences).
Cwnd convergence times with mixed RTT competition could get long, however.

One of the fundamental issues with AIMD, is that the additive term does not
scale to capacity (bandwidth).  Therefore, the amount of time it takes to return
to capacity after a congestion event increases proportionally with capacity.  We
experimented with using the advertised capacity in the network telemetry to
scale the additive increase term.  For example, at 100 Mbps, we add 1 MSS / RTT.
At 1 Gbps, we add 10 MSS / RTT.  This fixes the AIMD scaling problem, but
creates a new one: the induced queue length increases proportionally as well.
While the sojourn time stays the same, the queue length does not, and the
expectation in modern networking hardware is that queue lengths do **not**
scale with bandwidth.  It would be unreasonable to expect a 10000 packet queue
at 10 Gbps so that scaled AIMD congestion control mechanics work properly.

We also experimented with scaling the additive increase term with RTT.  High
RTTs are a problem for AIMD congestion control, because the recovery time after
a congestion event increases with the **square** of the RTT.  At geosat RTTs,
recovery times can be measured in minutes, and if any loss occurs, it's
catastrophic to throughput.  So, we scaled up the additive increase term with
RTT, and found that we had to additionally react with a greater multiplicative
decrease or else high RTT flows would dominate low RTT flows.  However, that
caused the problem of wild swings in cwnd (i.e. jitter) for high RTT flows.
There did not appear to be a good tradeoff to make here.

The above experiments with AIMD were what led to the idea for
[CSTA](../csta/csta.md), as we wanted something that would make a fundamental
break from AIMD's limitations.
