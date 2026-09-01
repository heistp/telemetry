<!--
SPDX-License-Identifier: GPL-3.0-or-later
Copyright 2026 Pete Heist
-->

# Congestion Control Telemetry Research

CSTA (Capacity Sharing with Time Accumulation) provides bottleneck capacity
sharing using a shared time accumulator in the network.  Please see
[CSTA](csta/csta.md) for details and simulation results.

[AIMD](aimd/aimd.md) contains a short writeup on some experiences using network
telemetry to enhance conventional AIMD congestion control.

The above two uses of network telemetry are at opposite ends of the spectrum.
AIMD takes a conventional approach, working with network hops that only send
network telemetry.  CSTA requires a feedback variable (the timeulator), a
significant coupling between endpoints and network hops that would lead to a
sea change in congestion control (and is also harder to deploy).  Further
research on options in-between would be worthwhile.
