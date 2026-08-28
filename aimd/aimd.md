# Telemetry Enhanced Additive Increase Multiplicative Decrease

- Compare with aqms and cubic/reno
- Discuss effect of changing MSS or growth per RTT
  - Higher MSS:
    - higher sojourn (not always)
    - faster processing
    - faster 1 MSS growth
  - Lower MSS:
    - lower sojourn (not always)
    - slower processing
    - slower 1 MSS growth
    - potential for cwnd instability at very high bandwidth
  - Problem is, there is no "right" growth size
    - Depends on expected number of flows in bottleneck
    - Depends on desired tradeoff between delay, FCT, convergence time
