# 4-day solo plan: feasibility pilot

Day 1 is the day you start. Nothing here has been run. Commands, image names and API paths below are from general knowledge and must be verified on your machine. Numbers marked *assumption* are guesses; replace them with measurements.

## What this can and cannot be

The full methodology (three scenarios, 30 blocks, independent rule interpreter, boundary suite) is not achievable solo in 4 days. This plan turns it into a **feasibility pilot**:

- **Can claim:** a working harness (or an honest account of where it broke), benign-concurrency trials, and how the mismatch-only check (B0) and the request-compatibility check (C1) behave on them, with small-sample intervals.
- **Can claim only if the injector works by end of day 2:** a handful of out-of-band attack trials (S2).
- **Cannot claim:** attack sensitivity if S2 isn't run, generalisation, anything about in-band attackers (that stays a logical argument), or hypotheses confirmed.

A good outcome is S1 (benign) trials running end to end, with B0 versus C1 compared on them.

## Scope

| Item | This plan |
|---|---|
| Testbed | 3-node ONOS 2.7.0 (Docker) in an Ubuntu Server 22.04 VM (12 GB RAM, 4 vCPUs, 50 GB disk) on a 16 GB host, with 3 Atomix + 3 ONOS containers, one OVS switch, two hosts. Verify each piece. |
| Scenarios | **S1 benign concurrency (main)**, S0 baseline (about 5 trials), **S2 only if the injector spike works** |
| Trials | 10–15 S1 trials per delay at 1–2 delays; about 5 S0; about 10 S2 if available. Set from measured trial time. |
| Evidence | Per-node flow state (REST), OVS flow dump, a simple live probe (delivered or not, with sequence numbers), proxy request log |
| Checks | B0 (any mismatch = alarm) and C1 (alarm only if a divergent rule has no matching logged request) |
| Validation | Hand-check about 10 trials against the request log instead of a held-out suite |
| Boundary controls | One matched benign/M2 record pair, done on paper or with synthetic records (no network run) |
| Dropped | Independent rule interpreter, held-out matcher suite, delay sweep, replay and delayed-install runs, artifact packaging |

## Day-by-day

### Day 1: environment and injector spike

- [ ] **(0:30–1:30) Linux environment:** Create an Ubuntu Server 22.04 VM on the 16 GB host: 12 GB RAM (3 Atomix + 3 ONOS means 6 JVMs), 4 vCPUs (3 if the CPU has fewer than 6 threads), 50 GB dynamically allocated disk. Check inside it: `sudo apt install openvswitch-switch`, `sudo modprobe openvswitch`, `sudo ovs-dpctl show`. These sizes are estimates. If you'd rather try WSL2 first, run the same three commands there and switch to the VM if any fails; don't build a custom WSL kernel. Time-box to 1.5 hours; last resort is OVS's user-space datapath, with the change stated in the paper.
- [ ] **(hard stop 2.5 h) Cluster:** Follow `day1_setup.md` Step 3. In ONOS 2.x this is 3 Atomix nodes plus 3 ONOS nodes. Cap container memory and Java heap, and watch `free -h` and `docker stats`. **Checkpoints:** at 45 min the 3 Atomix nodes are up and their logs show the cluster formed (if not, try the single-Atomix variant once, and state it in the paper); at 2 h `nodes` shows all 3 ONOS nodes READY; at 2.5 h stop, whatever the state. If the cluster isn't up at the hard stop, switch to the mechanism-model fallback below. The cluster step never rolls into Day 2.
- [ ] **(1 h) Reset script:** One command that tears everything down and recreates it. Time it.
- [ ] **(1.5 h, time-boxed) Injector spike:** Can an ONOS app on one node install a flow without going through REST, active only on that node (check the local node ID)? Yes or no by the end of the box. Start it only after the cluster checkpoints have passed; skip it if you hit the cluster hard stop.
- **Exit:** cluster up, reset scripted and timed, one flow seen in the OVS dump, injector answer recorded.

### Day 2: instruments and pilot trials

*If you took the mechanism-model fallback, Day 2 builds the model and checks instead; see the fallback section.*

- [ ] **(2 h) Logging proxy:** One small proxy per node (3 ports) writing JSON lines: node, timestamp, request body. This is your request log.
- [ ] **(2 h) State collector:** After each trial's wait, dump each node's flow state, the OVS flow table, and the probe result into one folder per trial.
- [ ] **(2 h) S1 driver and pilot:** Send two contradictory requests to two non-master nodes with a delay Δt; run about 10 trials across 2–3 delays. Define "divergent" simply: nodes disagree with each other, or the OVS table or probe disagrees with a node.
- **Decision at end of day 2:** if no divergence appears at any delay, stop collecting and write up the harness plus the negative result. If it appears, pick 1–2 delays and continue.
- **Exit:** per-trial folders, divergence seen or not, delays chosen.

### Day 3: freeze, collect, build the checks

- [ ] **(0:30) Freeze:** Git-tag the C1 rule, matching window, delays, trial counts and exclusions before looking at results.
- [ ] **(3–4 h) Collect:** S1 batches, about 5 S0, and S2 if the injector worked. Run these in the background.
- [ ] **(meanwhile) Build B0 and C1** over the trial folders. Hand-check about 10 trials against the proxy log.
- **Exit:** dataset collected, B0 and C1 outputs per trial.

### Day 4: analysis and paper

- [ ] **(2 h) Analysis:** Counts, divergence rate, B0 and C1 alarms on benign divergent trials, indeterminate cases, Wilson 95% intervals. Attack sensitivity only if S2 ran.
- [ ] **(3–4 h) Paper:** Switch the methodology to "feasibility pilot"; add results and limitations; list what was not run.
- [ ] **(1 h) Buffer and packaging:** Put scripts, configs and trial folders in one repo.

## If something breaks

| Problem | Response |
|---|---|
| No suitable Linux/OVS on Day 1 | Linux VM; else OVS user-space datapath with the change stated. |
| Cluster not up at the 2.5 h hard stop | Stop. Take the mechanism-model fallback below. Do not carry the cluster work into Day 2. |
| VM swaps or runs out of memory | Tighten the Java heap caps, close host apps, and as a last resort use one Atomix node (state it in the paper). If it still can't run stably, take the mechanism-model fallback. |
| Injector spike fails | Run S1 and S0 only. Report that S2 wasn't run; no attack sensitivity. Do not substitute a switch-level mutation. |
| No divergence in the pilot | Report the tested conditions and the interval; no attribution results. |
| Behind schedule | Cut in order: S2 trials, S0 trials, number of delays (to one), number of trials. Never cut the Day-3 freeze. |

## Fallback: mechanism model (if the cluster hits its hard stop)

Decide this now, not while frustrated. If the ONOS cluster isn't working at the hard stop, replace the testbed with a small Python model and say exactly that. The methodology requires a fallback to be described by what it actually models.

**What it is:** three replicas, each with its own store; asynchronous replication with a random delay; two contradictory requests sent to two non-master replicas with delay Δt; a request log that records requests at entry; an "unlogged addition" that writes into one replica's store with no log entry; and the same B0 and C1 checks over the resulting records. Parameters to state: delay distribution, log-loss probability, matching window W.

**What it can claim:** how the B0 and C1 logic behaves under the stated assumptions, including edge cases (a log gap, a delayed install, a replayed request, an in-band request that mimics a legitimate client).

**What it cannot claim:** any divergence rate, persistence, or attack sensitivity for ONOS. Those come from the model's parameters, so they are inputs, not findings. Results will follow largely from the assumptions, so the value is in exposing where the check logic breaks. Trial counts can be large because trials are cheap, but they are a simulation budget and not evidence about a real system.

**Schedule if you take it:** Day 2 builds the model and the checks; Day 3 freezes and runs them; Day 4 is analysis and paper. Anything you got working on a single ONOS node (proxy, request log, probe, reset) is partial harness work and goes in the paper as such.

**Paper wording:** call it a "mechanism model", not a testbed or an evaluation of ONOS.

## Run-time estimate (assumptions)

Trial time = reset + 60 s horizon. At a 2-minute reset, 25 trials take about 1.3 hours; at 6 minutes, about 2.9 hours. If your reset is slow, reduce the trial count, not the freeze.

Wilson 95% upper bound with zero events in n independent trials = z² / (n + z²), z = 1.96: n = 5 gives 43.4%, n = 10 gives 27.8%, n = 15 gives 20.4%. These samples are small; report intervals and do not generalise.

## If a teammate can give a few hours

Two tasks don't touch the testbed: (1) the analysis script (counts, Wilson intervals, B0/C1 tables) from a sample trial-folder format, and (2) the paper edits. The plan doesn't depend on either.

## Paper edits to make

- Introduction: describe this as a feasibility pilot, not an evaluation. Keep RQ but say the pilot tests whether the harness can answer it.
- Methodology: replace the 30-block, 7-trial design with this scope. Mark every unrun item as "not run" and keep the failed-gate language.
- Remove claims that depend on S2 if S2 isn't run. Keep in-band mimicry as a stated limitation argued logically.
- If you used the mechanism-model fallback: call it a mechanism model, report no ONOS divergence rates, and state its assumptions and parameters.
