# Day-1 setup guide

Time-box for the whole day is about 8–9 hours. Commands are written from the ONOS wiki notes I found and from general knowledge; I have **not** run any of them. Anything marked **(verify)** is a guess to confirm on your machine. Replace `<…>` placeholders.

**What the research found that changes the setup:** in ONOS 2.x an ONOS "cluster" is *two* clusters. ONOS nodes sit on top of a separate **Atomix** cluster that stores state and coordinates the nodes. The old `onos-form-cluster` script is no longer supported, and `cluster.json` now lists the Atomix nodes. So your cluster is **6 containers** (3 Atomix + 3 ONOS), not 3. The ONOS wiki page "Notes on cluster formation for Docker instances" is the closest guide, but it was written for ONOS 2.2.2 and Atomix 3.1.5, so adapt it.

## Step 0 (10 min): check for a ready-made setup

Open the GapFuzz arXiv page (2606.11035) and search the paper for a repository or artifact link. If they published a Docker setup, use it and skip steps 3–4. I could not read the paper body, so I don't know whether one exists.

## Step 1 (1–1.5 h): VM

- Ubuntu Server 22.04, 4 vCPUs, **12 GB RAM** (6 JVMs; start at 10 GB only if you want to test the limit), 50 GB dynamically allocated disk, SSH server enabled.
- **Take a VM snapshot** when the base OS is ready, and another after the cluster works. Snapshots are your safety net.

```bash
sudo apt update
sudo apt install -y openvswitch-switch mininet docker.io curl jq git python3-pip
sudo systemctl enable --now docker
sudo usermod -aG docker $USER      # log out and back in
sudo modprobe openvswitch
lsmod | grep openvswitch           # expect a line
sudo ovs-vsctl show
sudo ovs-dpctl show                # expect a kernel datapath listed
```

If `modprobe openvswitch` fails, you do not have the kernel datapath. Stop and decide (see the plan's fallback).

## Step 2 (30 min): find the right image versions

```bash
docker pull onosproject/onos:2.7.0                      # (verify the tag exists)
docker run -d --name onos-probe onosproject/onos:2.7.0
docker logs onos-probe 2>&1 | grep -i atomix | head     # note the Atomix version it reports
docker rm -f onos-probe
```

ONOS logs the Atomix version it expects, and the Atomix server version must match it. Pull the matching Atomix image (the wiki used `atomix/atomix:3.1.5`; the right tag for ONOS 2.7.0 is probably newer, **verify**).

## Step 3 (hard stop 2.5 h): the cluster (the risky step)

**Stop rules (decided in advance):**

- **45 min:** the 3 Atomix containers are up and their logs show the cluster formed. If not, try the single-Atomix variant once and note it for the paper.
- **2 h:** `nodes` in the ONOS CLI shows all 3 ONOS nodes READY.
- **2.5 h: hard stop.** If the cluster isn't up, stop and take the mechanism-model fallback in `plan.md`. Do not roll this step into Day 2.


Follow the ONOS wiki page "Notes on cluster formation for Docker instances" with your versions. In outline:

1. Run 3 Atomix containers (`docker run -t -d --name atomix-1 atomix/atomix:<version>`, and 2 and 3). Note each container IP with `docker inspect atomix-N | grep -i ipaddress`.
2. Generate the Atomix config files. The generator scripts live in `tools/test/bin/` of the ONOS source repo (check out tag 2.7.0 and list that folder; names from memory: `atomix-gen-config`, `onos-gen-config`, **verify**). You only need the repo for these scripts.
3. Copy each Atomix config into its container at `/opt/atomix/conf/atomix.conf`, then `docker restart atomix-N`.
4. Run 3 ONOS containers from `onosproject/onos:2.7.0`. Note their IPs.
5. Generate a `cluster.json` per ONOS node, pointing at the three Atomix nodes, then `docker exec onosN mkdir -p /root/onos/config` and `docker cp cluster-N.json onosN:/root/onos/config/cluster.json`, then restart the ONOS containers.
6. Two traps from people who hit this: the cluster name in `atomix.conf` **must equal** the `name` field in `cluster.json`, and `cluster.json` is read at runtime from the ONOS config directory, not at build time.

Memory: cap containers with `docker run --memory ...` and the Java options (check the ONOS image docs for the variable name, **verify**), and watch `free -h` and `docker stats`. If memory is very tight, a single Atomix node is a way to save about one JVM, but it differs from a replicated store and must be stated in the paper.

**Verify the cluster:**

```bash
ssh -p 8101 karaf@<onos1-ip>        # default password may be karaf, (verify)
onos> nodes                         # expect 3 ONOS nodes READY
onos> masters
curl -s -u onos:rocks http://<onos1-ip>:8181/onos/v1/cluster | jq .   # (verify credentials)
```

If the nodes can't see each other, read both the Atomix and ONOS logs, but only inside the stop rules above. A GitHub issue I found describes Docker Compose setups where none of the nodes could join, so don't assume it will just work.

## Step 4 (45 min): activate only what you need

Do **not** activate reactive forwarding (`org.onosproject.fwd`); it would install rules you didn't ask for. Activate the OpenFlow providers only (verify the app names in your ONOS version):

```bash
curl -u onos:rocks -X POST http://<onos1-ip>:8181/onos/v1/applications/org.onosproject.openflow/active
```

## Step 5 (45 min): switch and hosts with Mininet

Save as `topo.py` and run with `sudo python3 topo.py`. It uses OVS with the kernel datapath, two hosts, static ARP, and all three ONOS nodes as controllers.

```python
from mininet.net import Mininet
from mininet.node import RemoteController, OVSKernelSwitch
from mininet.topo import SingleSwitchTopo
from mininet.cli import CLI

net = Mininet(topo=SingleSwitchTopo(2), controller=None,
              switch=OVSKernelSwitch, autoSetMacs=True)
for i, ip in enumerate(['<onos1-ip>', '<onos2-ip>', '<onos3-ip>'], 1):
    net.addController(f'c{i}', controller=RemoteController, ip=ip, port=6653)
net.start()
net.staticArp()
CLI(net)
net.stop()
```

Check in the ONOS CLI that `devices` shows `of:0000000000000001`, and that `masters` shows which node is master for it. Clean up leftovers between runs with `sudo mn -c`.

## Step 6 (30 min): install a flow by REST and see it in OVS

With no reactive forwarding, `h1 ping h2` fails until you install rules. Install one per direction (port numbers assume h1 on 1 and h2 on 2; confirm in the ONOS CLI):

```bash
cat > flow12.json <<'EOF'
{"priority":40000,"timeout":0,"isPermanent":true,
 "deviceId":"of:0000000000000001",
 "treatment":{"instructions":[{"type":"OUTPUT","port":"2"}]},
 "selector":{"criteria":[{"type":"IN_PORT","port":"1"}]}}
EOF
curl -u onos:rocks -X POST -H 'Content-Type: application/json' \
  -d @flow12.json "http://<onos1-ip>:8181/onos/v1/flows/of:0000000000000001?appId=pilot"
```

Make the reverse rule (swap ports 1 and 2), then:

```bash
# in the Mininet CLI
mininet> h1 ping -c 3 h2
# in another shell
sudo ovs-ofctl -O OpenFlow13 dump-flows s1     # OpenFlow table
sudo ovs-dpctl dump-flows                      # kernel datapath cache
curl -s -u onos:rocks http://<onos2-ip>:8181/onos/v1/flows | jq .   # another node's view
```

Check the replica's view and the OVS tables agree. These three views are your evidence sources in the plan. Remember that the study treats the live ping as primary forwarding evidence and the dumps as diagnostic state.

## Step 7 (45 min): reset script

The plan requires a verified reset. Write `reset.sh` that:

1. Deletes your flows through REST (by `appId=pilot`) on one node.
2. Runs `sudo ovs-ofctl -O OpenFlow13 del-flows s1` and `sudo ovs-dpctl del-flows`.
3. Waits and **checks** all three nodes' flow lists and both OVS tables are empty (print a pass/fail line).

Time it. If leftovers persist (megaflows or stale flows on a node), fall back to restarting the ONOS and Atomix containers and Mininet, and time that too. Slow resets cut your trial count directly.

## Step 8 (90 min, time-boxed): injector spike

Start this only if Steps 3–7 passed.

The goal is an **unlogged internal install on one node**. Test options in this order and stop at the first that works:

1. **Node-local ONOS app** that checks the local node id and installs a flow only there. Cleanest match to the methodology, but it needs a working ONOS app build (Maven or Bazel) and is the likeliest to blow the time-box.
2. **Cheaper stand-in:** use the ONOS CLI on one node (SSH to port 8101) to add an intent or flow (`add-point-intent` or similar, **verify** it exists in 2.7.0). It goes through the node's own services and not through the REST proxy, so the proxy log never sees it. This **emulates** an unlogged internal install; it is not a compromised replica. If you use it, say so in the paper and do not call it equivalent.

Whatever you use, confirm independently that it acted on exactly one node (switch-side evidence and per-node flow lists), not just from its own logs.

## End-of-day checklist

- [ ] Kernel datapath confirmed (`ovs-dpctl show`)
- [ ] 3 ONOS nodes + 3 Atomix nodes up, `nodes` shows 3 READY
- [ ] Switch connected, masters visible
- [ ] Flow installed via REST and seen on a second node and in both OVS tables
- [ ] `h1 ping h2` works only after the rules exist
- [ ] Reset script written, timed, and verified
- [ ] Injector: yes / stand-in only / no (recorded)
- [ ] VM snapshot taken; commands and versions logged in the repo
