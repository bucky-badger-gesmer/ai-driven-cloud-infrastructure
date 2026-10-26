# BARO Paper Walkthrough

A study guide for Pham, Ha, and Zhang, "BARO: Robust Root Cause Analysis for Microservices via Multivariate Bayesian Online Change Point Detection," Proc. ACM Softw. Eng. 1(FSE), Article 98, 2024. [doi:10.1145/3660805](https://doi.org/10.1145/3660805). The PDF is `3660805.pdf` in this folder.

Read it top to bottom once. Before presenting, reread sections 1, 6, 9, and 10.

---

## 1. The paper in one minute

When something breaks in a microservice system, operations does two things:

1. **Detect:** notice that something is wrong, and roughly when it started.
2. **Diagnose (root cause analysis, RCA):** work out which service, and which metric on it, is to blame.

Most RCA research only does step 2. It assumes step 1 already produced an accurate failure start time. In practice detectors are noisy, so the start time is often early or late. The authors' point is that many RCA methods quietly fall apart when that start time is wrong.

BARO is an end-to-end pipeline with two modules:

- **Multivariate BOCPD** (Bayesian Online Change Point Detection) watches latency and error metrics together and reports the first moment their joint behavior changes. That moment is the estimated failure start.
- **RobustScorer** takes that start time, learns what "normal" looked like before it using the **median and interquartile range (IQR)**, and ranks every metric by how far it moved afterward. The top of the list is where the operator looks first.

The key idea is that median and IQR barely move when a few broken data points leak into the "normal" window. Mean and standard deviation move a lot. So BARO's ranking still works when the start time is a bit off.

**Three contributions the paper claims:**

1. The end-to-end method (the two modules above).
2. Experiments on three benchmark systems, where BARO beats the baselines at both detection and RCA.
3. A sensitivity analysis showing BARO is much less sensitive to an inaccurate start time than the baselines.

---

## 2. Vocabulary you need

| Term | Plain meaning |
|---|---|
| **Fault** | The underlying cause. Examples: a CPU hog, a memory leak, network delay. |
| **Failure** | The service not doing its job (slow, erroring). |
| **Anomaly** | The visible symptom of a failure in the metrics. |
| **RCA** | Root cause analysis: working out why the failure happened. |
| **Metric** | One time series from monitoring, such as `carts_cpu` or `frontend_latency`. |
| **Multivariate time series** | Many metrics recorded side by side over the same time steps. |
| **Four golden signals** | Traffic, Saturation, Latency, Errors, from Google's SRE book. BARO uses these to decide what counts as a failure. |
| **Change point** | A time where a series' statistical behavior shifts. |
| **Run length** | How many points in a row have come from the same "regime" since the last change point. This is what BOCPD tracks. |
| **z-score** | (value minus mean) divided by standard deviation. N-Sigma and CIRCA rank with this. |
| **Median / IQR** | The middle value / the spread of the middle 50% (75th percentile minus 25th). They are "robust": outliers barely move them. |
| **Coarse-grained RCA** | Naming the right **service**. |
| **Fine-grained RCA** | Naming the right **metric** (for example, memory on `carts`). |
| **AC@k** | Fraction of cases where the true root cause is in the top k of the ranked list. |
| **Avg@5** | The average of AC@1, AC@2, AC@3, AC@4, AC@5. It rewards putting the right answer near the top, not just anywhere in the top five. |
| **Precision / Recall / F1** | Detection scores. Recall: of the real failures, how many were caught. Precision: of the alarms raised, how many were real. F1: a single number combining the two. |
| **t_inject** | The time the fault was actually injected. The paper uses it as the "true" start time. |
| **t_bias** | How far a deliberately shifted start time is from t_inject. Negative = early, positive = late. |

---

## 3. The problem, and why it matters (paper Sections 1 and 2)

**Why anyone cares:** failures in one service spread to others. The paper cites an estimate that an hour of Amazon.com downtime could cost up to US$100 million. Operators need to find the failure quickly and know where to look.

**The usual pipeline:**

```
metrics  ->  anomaly detector  ->  (start time)  ->  RCA method  ->  ranked suspects
```

**The gap:** RCA papers treat the detector as solved. Some RCA methods, such as CIRCA and RCD, explicitly need the failure start time and assume it is accurate. Meanwhile, the detectors these papers pair themselves with are simple ones:

- **N-Sigma** (three-sigma rule): anything more than 3 standard deviations from the mean is abnormal. Used by MicroScope and MS-Rank.
- **SPOT / dSPOT:** extreme value theory thresholds.
- **BIRCH:** a clustering algorithm. Used by MicroRCA and MicroDiag.
- **Univariate Bayesian change point detection (UniBCP):** used by CauseInfer, one metric at a time, offline.

Commercial tools such as Datadog and Dynatrace also mostly use univariate threshold detectors, similar in spirit to N-Sigma.

So nobody had checked: **if the start time from a realistic detector is a little wrong, does the RCA still work?**

### The three families of RCA methods (paper Section 2.4)

You don't need to know these in depth. Know the buckets:

1. **Statistical:** compare metrics before and after the start time. Examples: N-Sigma (z-score), ε-Diagnosis (two-sample test). BARO is in this family.
2. **Topology graph:** build a graph of which service calls which, then search it. Examples: MicroRCA (random walk), CIRCA (hypothesis tests on a structural graph built from a **call graph**).
3. **Causal graph:** learn cause-and-effect edges between metrics from the data, then rank with PageRank, random walk, etc. Examples: RCD (divide and conquer with Ψ-PC), CausalRCA (DAG-GNN + PageRank).

Graph methods need a graph. On a large system that graph is expensive to learn, or the learning fails. That shows up in the results on Train Ticket.

---

## 4. How BARO works (paper Section 3)

### 4.1 Two assumptions everything rests on (Section 3.1)

**Assumption 1: A failure shows up in Latency or Errors.**
A spike in traffic or CPU (for example, holiday load) with **no** rise in latency or errors is treated as normal. A spike that **does** push latency or errors up counts as an anomaly. Detection only looks at latency and error metrics. Ranking looks at **all** metrics.

> Consequence to remember: if a fault never moves latency or errors, BARO will not detect it.

**Assumption 2: The first anomaly marks when the failure started.**
A failure spreads: the faulty service misbehaves first, then its callers, then theirs. So the abnormal period contains several change points. BARO takes the **first** one as the start time and uses it to split the data into "normal" (before) and "abnormal" (after).

> This does **not** mean the first anomaly is the root cause. It is only a clock. The paper says this explicitly, and it is a common point of confusion.

### 4.2 Module 1: Multivariate BOCPD (Section 3.3)

**Intuition for BOCPD.** Imagine reading a time series one point at a time. After each point you ask: "Does this new point look like it came from the same behavior as the recent run, or has something changed?" BOCPD keeps a probability for every possible **run length** (0, 1, 2, ... points since the last change). If the new point fits, the most likely run length grows by one. If it doesn't, the probability mass shifts to run length 0. A drop in the most likely run length is a change point.

- **Bayesian:** it combines a prior (how often changes tend to happen, from a "hazard function") with how well new data fits.
- **Online:** it processes points as they arrive, rather than looking at the whole series after the fact.
- **No user threshold:** you don't pick "3 sigma" or similar for each metric.

**What "multivariate" adds.** Instead of running BOCPD separately on each metric, BARO models the latency and error metrics **together** as one multivariate Gaussian (using a method from Xuan and Murphy, 2007). That means it can catch a change in **how the metrics move relative to each other** (their correlation structure), not just a jump in one line. Because failures propagate, several metrics tend to shift together, so a joint model fits the problem.

You do **not** need to derive Equations 1 and 2. If asked: Eq. 1 is the recursive Bayes update for run length; Eq. 2 is the marginal likelihood for a multivariate Gaussian with an inverse-Wishart prior on its covariance. That likelihood replaces the univariate one in Eq. 1.

**Output:** y = 1 (anomaly) plus the estimated start time t̂_A, or y = 0 (normal).

### 4.3 Module 2: RobustScorer (Section 3.4)

For **every** metric:

1. **Learn the normal band.** Take the data from the start of the window up to t̂_A. Compute the **median** and **IQR**.
2. **Score each later point.** For each point after t̂_A:
   `score = |value - median| / IQR`
3. **Keep the worst.** The metric's final score ρ is its **maximum** score over the abnormal period.

Then sort all metrics by ρ, highest first. That is the fine-grained ranking (metrics). Read off which service each metric belongs to and you have the coarse-grained ranking (services).

This is essentially a "robust z-score": the same shape as a z-score, with median in place of mean and IQR in place of standard deviation.

**Worked example (made-up numbers).** A metric's normal values are about 10 to 12. The failure pushes it to 30 and above.

| Normal window used | Mean | Std dev | Median | IQR | z-score of a value of 34 | RobustScorer score of 34 |
|---|---|---|---|---|---|---|
| Correct start time (10 clean points) | 10.8 | 0.75 | 11 | 1 | **31.0** | **23.0** |
| Late start time (same 10 points + 2 broken ones: 30, 32) | 14.2 | 7.57 | 11 | 2 | **2.6** | **11.5** |

With a late start, two broken points leak into "normal." The standard deviation jumps tenfold and the z-score collapses from 31 to 2.6. That's low enough that ordinary noise on a healthy metric could outrank it. The median doesn't move at all, and the robust score only halves. The guilty metric stays near the top.

That is the paper's whole robustness argument in one table. The paper's Fig. 3 shows the same thing with real data: with a late start, mean and sd shift from 0.23/0.20 to 0.27/0.24, while median and IQR only shift from 0.17/0.17 to 0.19/0.19.

**Early detection** is the other case (Fig. 3a). Then the normal window is just shorter, with fewer points to learn from. Median and IQR also hold up better than mean and sd with little data.

**"Hypothesis test" but no p-value.** The paper frames RobustScorer as a nonparametric hypothesis test (null hypothesis: "this metric is not a root cause"). But BARO never computes a p-value or accepts/rejects anything. It only uses the score to **rank**, because operators work down a list from the top. If someone asks, "is it really a hypothesis test?", the honest answer is that it is a robust deviation score used for ranking, framed as a test.

**Other properties the paper claims:** distribution-free (no assumption that data is Gaussian), scale-equivalent (a metric in milliseconds and one in percent are comparable because each is divided by its own IQR), and "rotation-invariant," which the paper defines as timestamp shifts not affecting the ranking.

**Correlated failures (Section 3.4.3).** If one hidden cause (say, a firewall misconfiguration) breaks services A and B at once, RobustScorer puts both A and B at the top. The operator then checks those two. BARO never names the firewall itself, because that isn't a metric it has.

### 4.4 Algorithm 1 in plain words

```
BARO(all metrics):
    y, start = MultivariateBOCPD(latency + error metrics only)
        - compute run-length probabilities over time
        - first time the most likely run length drops -> anomaly at that time
        - no drop -> normal, stop
    if anomaly:
        return RobustScorer(all metrics, start)
            - for each metric: median, IQR from before start
            - score = max over after-start points of |x - median| / IQR
            - sort metrics by score, descending
```

---

## 5. The experiment (paper Section 4.1 to 4.4)

**Cluster:** real Kubernetes, 1 master + 5 workers, each worker 16 CPUs / 32 GB (80 CPUs / 160 GB total). One replica per service. Load: 100 to 200 concurrent users, about 40 to 50 requests per second.

**Monitoring:** Istio service mesh + Prometheus + cAdvisor. Both resource metrics (CPU, memory) and service metrics (latency, errors, request counts).

**Three benchmark systems:**

| System | Services | Metrics | What it is |
|---|---|---|---|
| Online Boutique | 12 | 49 | Google's e-commerce demo |
| Sock Shop | 11 | 46 | Weaveworks' sock store demo |
| Train Ticket | 64 | 212 | Fudan's train booking system, much longer failure chains |

**Faults:** 4 types, injected with `kubectl exec` into the container:

- CPU hog and memory leak with `stress-ng`
- Network delay and packet loss with `tc` (Linux traffic control)

5 target services per system, 4 fault types, 5 repeats each = **100 failure cases per system**.

**Two details that matter later:**

- The services are **restarted after each injection** to keep runs independent, rather than waiting for a cooldown.
- Visual inspection showed symptoms appear **1 to 3 seconds** after injection. That's why t_inject is a fair stand-in for the true start time.

**Each method is run 5 times per case and averaged.** Hyperparameters for baselines come from their papers or public code.

**Baselines:**

- Detection: N-Sigma, SPOT, BIRCH, UniBCP.
- RCA: Dummy (random pick), CausalRCA, ε-Diagnosis, RCD, CIRCA, N-Sigma.
- Each RCA method is paired with every detector, written `Method[Detector]`. For example, `RCD[SPOT]` means RCD fed the start time from SPOT. `Method[t_inject]` means it was given the true time (the "ideal" upper bound). `BARO` = `RobustScorer[BOCPD]`.
- CIRCA normally needs a real call graph. None was available, so the authors built its graph with the PC causal discovery algorithm instead. Keep that in mind when CIRCA looks bad.

---

## 6. Results, one research question at a time (Sections 4.5 to 4.9)

### RQ1: Is BARO a better detector? (Table 2)

| Detector | Boutique F1 | Sock Shop F1 | Train Ticket F1 |
|---|---|---|---|
| N-Sigma | 0.70 | 0.72 | 0.67 |
| SPOT | 0.69 | 0.69 | 0.67 |
| BIRCH | 0.40 | 0.05 | 0.39 |
| UniBCP | 0.67 | 0.66 | 0.67 |
| **BARO** | **0.82** | **0.75** | **0.81** |

How to read it:

- **Every detector except BIRCH has recall near 1.0.** They catch every failure. The difference is **precision**: they also raise alarms on normal data. Baseline precision is about 0.50 to 0.56. BARO's is 0.60 to 0.69.
- So BARO's detection win is **fewer false alarms**, not catching more failures. It still has false alarms.
- **BIRCH barely fires** (Sock Shop recall 0.05). Any RCA result paired with BIRCH only covers the few cases BIRCH caught.

### RQ2: Is BARO better at finding the root cause? (Tables 3 and 4)

**Coarse-grained (service), overall Avg@5:**

| Method | Boutique | Sock Shop | Train Ticket |
|---|---|---|---|
| Dummy (random) | 0.25 | 0.38 | 0.07 |
| CausalRCA | 0.80 | 0.60 | **0.28** |
| N-Sigma[N-Sigma] | 0.74 | 0.83 | 0.75 |
| **BARO** | **0.86** | **0.95** | **0.81** |

- The headline is **Train Ticket: CausalRCA 0.28 vs BARO 0.81** (the paper calls this a 189% improvement). On the small Boutique system they're close (0.80 vs 0.86).
- RCD and ε-Diagnosis are near random on Train Ticket. RCD drops to about 0.05 to 0.08.
- **N-Sigma, a plain z-score, beats the fancy graph methods** (CIRCA and RCD) on service-level ranking. The paper says so.
- CIRCA on Train Ticket ran on only **15 of 100 cases**, because the PC algorithm failed to build its graph on the rest.

**Fine-grained (metric), overall Avg@5:**
BARO gets **0.61 / 0.81 / 0.67**. The best of any method in the table is **0.65 / 0.84 / 0.68**. So BARO is "among the top," not the clear winner, at naming the exact metric. It is far ahead of RCD (0.21 / 0.09 / 0.02 with the true time) and ahead of CIRCA.

### RQ3: Do both modules matter? (Tables 3 and 4 again)

- **Given the true start time, the simple methods tie.** Service Avg@5 with t_inject: CIRCA 0.77 / 0.95 / 0.70, N-Sigma 0.86 / 0.96 / 0.77, RobustScorer 0.86 / 0.95 / 0.79. With a perfect clock, N-Sigma is as good as BARO's scorer (and slightly better on Sock Shop: 0.96 vs 0.95).
- **With a real detector, they separate.** Across detectors on Boutique, CIRCA's score varies by 62%, N-Sigma's by 33%, RobustScorer's by 25%.
- **Swap only the scorer.** Keep BARO's BOCPD detector and rank with z-scores instead (`N-Sigma[BOCPD]`): service Avg@5 is **0.54 / 0.76 / 0.30**. With RobustScorer (that's BARO): **0.86 / 0.95 / 0.81**. On Train Ticket, the scorer alone is the difference between 0.30 and 0.81.
- Multivariate BOCPD is also the best detector to pair with RobustScorer: it gets close to the t_inject result.

### RQ4: How sensitive is each method? (Figs. 5 and 6)

**Start time sensitivity (Fig. 5, Online Boutique; Sock Shop in the supplement):** the authors shift the start time from 40 before to 40 after t_inject (the paper doesn't state the unit) and re-run the RCA.

- **BARO stays flat** across the whole range.
- **ε-Diagnosis is also flat, but at a low score.** Being robust and being accurate are different things.
- **N-Sigma and CIRCA drop when the start is late**, because both use mean and sd (z-scores), which outliers in the "normal" window drag around. This is the Fig. 3 effect showing up in real results.
- **RCD** is also sensitive.

**Hyperparameter sensitivity (Fig. 6):** CIRCA's α, RCD's chunk size γ, and ε-Diagnosis's α barely change their results. This tells you the baselines weren't handicapped by bad settings, which makes the comparison fairer.

### Running time (Tables 5 and 6)

- **RobustScorer:** about **0.01 s** per case. At least 3 to 1000 times faster than the other RCA methods.
- **BARO's detector:** average **30 to 173 s** per case depending on system. Worst single case: about **7 minutes** on Train Ticket.
- For comparison: CIRCA took about 7,565 s per case on Train Ticket, CausalRCA about 2,639 s, UniBCP's detection 638 to 3,292 s.
- Experiments ran on 8-CPU / 16 GB Linux servers, Python 3.10.

So the ranking is effectively instant. The detector is the slow part. It's fine for batch triage, but too slow to run on every Prometheus scrape as-is.

---

## 7. Limitations: theirs and yours

**What the paper itself admits (Section 5, Threats to Validity):**

- Only four fault types.
- One deployment setting and one data collection strategy per system.
- BARO only applies where its assumptions hold (failures show up in latency or errors).
- Future work: add logs and traces (multimodal data). Possibly other system types, such as distributed databases.

**What you can add (and the deck does):**

- The advantage is specifically about an **imprecise clock**. Given the true time, N-Sigma ties.
- Metric-level ranking is a close race, not a clear win.
- Runs are **restarted clean** between faults, so there's no slow drift, no long-lived cluster, and no overlapping incidents.
- All faults are **synthetic** and injected one at a time.
- The CIRCA comparison isn't fully fair: it ran on a guessed graph, not the call graph it was designed for.
- BIRCH-paired numbers cover only a subset of cases.
- The detector's runtime (up to 7 minutes) matters in a real incident.
- Correlated failures are ranked as several services near the top, but the output doesn't explicitly say "these moved together."

---

## 8. Map from your deck to the paper

| Slide | Topic | Where it comes from |
|---|---|---|
| 1 | Title, claim | Abstract |
| 2 | Why I chose it | Sec. 1 (the assumed-start-time gap); Sec. 7 (public code and data); the "Results Validated" badge on page 1 |
| 3 | The gap | Sec. 1, 2.3, 2.4; vocabulary from 2.1.1 |
| 4 | Pipeline | Fig. 1, Sec. 3.1 to 3.2, Algorithm 1 |
| 5 | Module 1 | Sec. 3.3, Fig. 2; latency and errors rule from 3.1.1 |
| 6 | Module 2 | Sec. 3.4, Eq. 3 to 4, Fig. 3b |
| 7 | Experiment | Sec. 4.1, Table 1 |
| 8 | Key results | Table 2 (F1), Table 3 (service Avg@5) |
| 9 | Robustness | Fig. 5; Table 3 `N-Sigma[BOCPD]` vs BARO |
| 10 | Caveats | Table 4 (metric Avg@5), Table 3 and 4 `N-Sigma[t_inject]` rows, Table 3 footnote (CIRCA 15/100, BIRCH) |
| 11 | Reflection | Sec. 4.9 (runtime), Sec. 5 (threats), Sec. 6 (future work) |
| 12 | Claim + citation | Sec. 6 |

---

## 9. Questions you might get, with short answers

**Q: Isn't BARO just a z-score with the median?**
Largely, yes, for the ranking step. That's the point: a small, cheap change in statistic makes ranking robust to a wrong start time. The other half is the multivariate detector, which reduces false alarms and gives a better start time.

**Q: If N-Sigma ties BARO with the true time, why not use N-Sigma?**
In real operations you never have the true time. You have a detector's guess. With a guess, N-Sigma degrades (Fig. 5, and 0.30 vs 0.81 on Train Ticket with the same detector).

**Q: Does BARO find the actual root cause?**
It ranks metrics and services. The paper reports how often the true one is near the top (Avg@5). An operator still investigates. It's a worklist, not an automatic fix.

**Q: Why is the first anomaly not the root cause?**
The first anomaly is just the earliest time anything in latency or errors changed. The root cause service might show up on a resource metric (say CPU) that isn't used for detection. Detection gives the clock; ranking gives the suspect.

**Q: What if a fault doesn't affect latency or errors?**
BARO won't detect it. That's a stated assumption, shared with prior metric-based RCA work.

**Q: Why is Train Ticket so important?**
It's the only large system (64 services, 212 metrics) with long failure chains. That's where graph-based methods break down (CausalRCA 0.28, RCD near random, CIRCA's graph failing 85% of the time) and BARO holds at 0.81.

**Q: How fast is it?**
Ranking: about 0.01 s. Detection: 30 to 173 s on average, up to about 7 minutes.

**Q: Precision is only 0.6 to 0.69. Is that good?**
It's better than every baseline (about 0.50 to 0.56), but it still means a meaningful share of false alarms. Recall is 1.0, so nothing was missed.

**Q: Can I trust the results?**
Code and datasets are public (GitHub and Zenodo), and the paper carries ACM's "Results Validated" artifact badge. Limits: synthetic faults, one cluster, restarts between runs.

**Q: How would you extend it?**
Logs and traces (the authors' own plan); an explicit "these services moved together" output for correlated failures; testing on a long-running cluster without restarts; a faster detector for real-time use.

---

## 10. Self-check (cover the answers)

1. What are BARO's two modules, and what does each output?
   *Multivariate BOCPD outputs whether there's an anomaly and when it started. RobustScorer outputs a ranked list of metrics (and therefore services).*
2. Which metrics does detection use? Which does ranking use?
   *Detection: latency and errors only. Ranking: all metrics.*
3. Why median and IQR instead of mean and standard deviation?
   *Outliers from a late start time (or a short window from an early one) barely move median and IQR.*
4. What does Avg@5 measure?
   *The average of AC@1 to AC@5, so ranking the right answer first scores higher than ranking it fifth.*
5. BARO vs CausalRCA on Train Ticket (service level)?
   *0.81 vs 0.28.*
6. What happens to the z-score ranker when you keep BARO's detector?
   *0.54 / 0.76 / 0.30, vs BARO's 0.86 / 0.95 / 0.81.*
7. Given the true start time, who wins?
   *Essentially a tie. N-Sigma 0.96 vs BARO 0.95 on Sock Shop service-level.*
8. Name two limits of the experimental setup.
   *Restarts between runs; four synthetic faults; one cluster; metrics only.*
9. Why did CIRCA run on only 15 Train Ticket cases?
   *The PC algorithm failed to build its causal graph on the other 85.*
10. How long does BARO take?
    *Ranking about 0.01 s; detection 30 to 173 s on average, worst about 7 minutes.*

---

## Reference

Pham, L., Ha, H., & Zhang, H. (2024). BARO: Robust root cause analysis for microservices via multivariate Bayesian online change point detection. *Proceedings of the ACM on Software Engineering, 1*(FSE), Article 98. https://doi.org/10.1145/3660805

Code: https://github.com/phamquiluan/baro
