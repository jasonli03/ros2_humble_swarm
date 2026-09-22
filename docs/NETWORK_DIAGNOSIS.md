# RoboMaster Swarm — Network Diagnosis and Access-Point Validation Procedure

**Date:** 2026-09-19
**Verdict:** The lab access point (NETGEAR WNDR3700v4, 2013) is the root cause of
the unstable RoboMaster driver connections. No code change is needed. Replace the
access point.

> **Terminology.** "Access point" always means the infrastructure WiFi device (the
> router the robots connect *to*). The robot's own two modes are called
> **networking mode** (robot joins the access point as a client — the normal swarm
> setup) and **direct-connect mode** (robot broadcasts its own hotspot and the host
> joins it — a diagnostic tool only). These match the labels on the robot's physical
> switch. Note the SDK's parameter names differ: `conn_type:=sta` is networking mode
> and `conn_type:=ap` is direct-connect mode.

---

## 1. Symptom

`ros2 launch swarm_bringup launch_centralized_swarm.py` fails to connect to the
robots. The driver logs `RECV TimeOut` on every SDK handshake, or connects and then
reports `Disconnected` almost immediately.

## 2. Root cause

The access point reports the robot as associated and ARP resolves its MAC, yet 50–90% of
packets never arrive. Association works; data delivery does not.

Measured on the **same robot** (`48:1c:b9:62:29:c7`, an S1) in the **same networking
mode**, changing only the access point:

| Access point | Packet loss | RTT min / avg / max |
|---|---|---|
| NETGEAR48 (WNDR3700v4) | 45% / 60% / 77.5% | 40 / 330 / 619 ms |
| iPhone personal hotspot | 5% / 2.5% / 5% / 8% (220 pkts) | **6.7** / 38–49 / 308–374 ms |
| None — robot in direct-connect mode | 5% / 7.5% / 10% | 3.8 / 19–35 / 166 ms |

The signature of the fault is the **minimum RTT floor**: ~40 ms on the WNDR3700v4
versus 6.7 ms elsewhere. Every single packet is delayed, and losses arrive in runs
of 3–10 consecutive packets (multi-second blackouts), not as random single drops.

End-to-end driver behaviour, same code, both in networking mode (`conn_type=sta`),
minutes apart:

| Target | Result |
|---|---|
| Robot on NETGEAR48 | **10/10 handshakes `RECV TimeOut`**, never connected |
| Robot on phone hotspot | **Connected in 239 ms on attempt 1/5**, ran 73 s with zero disconnects, heartbeat mean 1.00 s |

## 3. Eliminated — do not re-test these

| Layer | How it was eliminated |
|---|---|
| Host NIC, USB WiFi dongle, driver, wireless stack, association state | Cabled `eno1` to a LAN port and bypassed host WiFi entirely: gateway 0% / 0.3 ms while robot on the *same wire* was 52.5% |
| Distance / obstruction | Moving the robot to 1–2 m from the access point made it **worse** (30% → 65%) |
| Single external interferer | The two robots' blackouts are statistically independent |
| Per-unit robot hardware | A second robot behaved identically |
| Robot RF hardware, antenna, TX power | Direct-connect and phone-hotspot tests are clean on the same unit |
| Battery level | Same battery, same session, clean in direct-connect mode |
| 2.4 GHz channel congestion | Channel `Auto(6)` → fixed `11` produced no change (47.5 / 42.5 / 57.5%) |
| Channel width | `Up to 300 Mbps` (40 MHz) → `Up to 130 Mbps` (20 MHz), no change |
| Access point transmit power | Already 100% |
| Phone-app contention | Error was always `RECV TimeOut`, never `reject connection, service is busy` |
| ROS 2 / robomaster SDK software stack | Connects in 239 ms and runs clean on a good link |

### Known driver defects — amplifiers, not the cause
These are real and worth fixing eventually, but none of them fired on a good link:
- `conn.switch_remote_route` sends the handshake as a single datagram with **no retry**
- The reconnect loop in `robomaster_driver.py` sleeps a flat 0.1 s — **no backoff**
- `client.py:211` creates the 5 s `heartbeat_check` timer **before** ~33–49 s of blocking module init
- `robot.initialize()` returns `True` unconditionally, so `_connect_with_retries` cannot detect a useless connection

## 4. Measurement caveat

**ICMP is pessimistic on these robots.** With 486 msg/s of SDK data flowing perfectly,
ping still showed 10% loss. SDK traffic load did **not** change ICMP loss
(5–10% idle vs 10% loaded). All ping numbers above therefore understate real link
quality — but the gap between the old access point (50–90%) and the phone hotspot
(5%) is far too large for this to matter.

---

## 5. Validation procedure for the new access point

### 5.0 Prerequisite — cable the host's ethernet port

The host (**Dell Precision T1700**) has only ONE WiFi adapter. Switching it to a
robot hotspot severs all other connectivity. Cable the onboard NIC `eno1`
(RJ45 on the back I/O panel, PCI `0000:00:19.0`, MAC `98:90:96:e2:82:1d`) to a
**LAN** port on the access point — not the WAN/Internet port.

```bash
cat /sys/class/net/eno1/carrier   # must be 1
ip -br addr show eno1             # must have a 192.168.x.x address
```

Measuring from `eno1` keeps the host's WiFi out of the path entirely, which is what
made this diagnosis conclusive. Use it for every robot measurement.

> `iw` and `ethtool` are NOT installed on this host. Use `nmcli` and `/sys/class/net/*`.
> `sudo` is password-gated, so `tcpdump` needs a human at the keyboard.

### 5.1 What the new access point must support

Hard requirements — if any one of these is missing, the replacement can reproduce the
same fault:

- **A dedicated 2.4 GHz SSID**, i.e. band steering / "Smart Connect" must be
  *disableable*. The robots are 2.4 GHz only.
- **WPA2-PSK (AES) selectable on its own** — not locked to WPA3 or to a WPA2/WPA3
  mixed mode.
- **PMF / 802.11w settable to Disable (or Capable)**, not forced Required.
- **Fixed channel selection** (no forced auto-channel).
- **Client isolation off, and broadcast/multicast forwarded.** SN discovery uses UDP
  broadcast port 40927; ROS 2 DDS discovery uses multicast.
- **DHCP address reservation** per MAC, so robot IPs stop moving.

Strongly recommended:

- **DTIM / beacon interval adjustable** (the old access point's successful RTTs
  clustered at ≈ beacon 100 ms × DTIM 3).
- **802.11r / k / v individually disableable.**
- **No minimum data rate and no minimum-RSSI kick** — or at least the ability to
  turn them off.
- **A client list showing association state and RSSI**, which is how the "associated
  but ARP-unreachable" signature was spotted.

Practical note: prefer a **router with 4 LAN ports** over a ceiling-mount access
point. §5.0 requires cabling the host's `eno1`, and a PoE ceiling unit has a single
uplink port, so it would need an extra switch.

Note that raw throughput is *not* the requirement. An iPhone personal hotspot — a
tiny, low-power radio — outperformed the lab router by a factor of ten. What is
needed is a modern radio plus configurability, not more power.

#### Candidate models

| Model | Notes |
|---|---|
| **ASUS RT-BE58U** (recommended) | 4× gigabit LAN + 1× 2.5G WAN/LAN, longer official firmware support runway. Not supported by Asuswrt-Merlin as of 2026-04. |
| ASUS RT-AX58U **V1 only** | Cheaper; Asuswrt-Merlin supported (SSH, wireless logs). **V2 is not Merlin-supported** — check the hardware revision on the box before buying. |

Avoid mesh kits: band steering and the merged SSID are usually not disableable.

### 5.2 Configure the new access point

Apply these before testing. The RoboMasters are **2.4 GHz only** and have old WiFi firmware.

- **Dedicated 2.4 GHz SSID**; disable band steering (otherwise the access point tries
  to push 2.4-only clients to 5 GHz)
- **PMF / 802.11w → disabled or optional.** A robot failed to associate to an iPhone
  hotspot, most likely for this reason
- **Disable 802.11r / k / v** (fast roaming confuses legacy clients)
- **DTIM = 1** (successful RTTs clustered at 310/312/320 ms ≈ beacon 100 ms × DTIM 3)
- **Keep 802.11b/g legacy rates**; do not set a minimum data rate or minimum-RSSI kick
- **Disable client isolation**
- **Confirm broadcast and multicast forwarding.** SN discovery uses UDP broadcast
  port 40927; ROS 2 DDS discovery uses multicast
- Security **WPA2-PSK (AES)** — not WPA3, not WPA/WPA2 mixed, not TKIP
- Fixed channel (11 was cleanest in this lab: only 3 co-channel neighbours at signal
  52–65; channel 6 had 4 co-channel plus 2 overlapping on channel 9)

#### ASUSWRT unboxing checklist

If the replacement is one of the ASUS models above, this is the exact click-path.
Apply all of it **before** connecting any robot.

**Wireless → General**
- **Smart Connect → OFF** (this is what splits the bands into separate SSIDs)
- Band **2.4 GHz**, give it its own SSID
- Authentication Method → **WPA2-Personal** (*not* WPA2/WPA3-Auto)
- WPA Encryption → **AES**
- **Protected Management Frames → Disable**
- Control Channel → **11** (not Auto); Channel bandwidth 20 MHz

**Wireless → Professional** (band = 2.4 GHz)
- **DTIM Interval → 1**
- Beacon Interval → 100
- **Roaming Assistant → Disable** — this is ASUS's minimum-RSSI kick and it targets
  exactly the kind of weak legacy client the robots are
- Disable 802.11k / 802.11v (and 802.11r under Wireless → Roaming Block List /
  General, depending on firmware version)
- Leave legacy b/g rates enabled; do not set a minimum data rate

**LAN → DHCP Server**
- Add a **manual IP assignment** for each of the six robots (MAC → IP)
- Then update `robot_ips` in
  `ros2_swarm/src/swarm_bringup/launch/launch_centralized_swarm.py:36` to match

**LAN → LAN IP** — ASUS defaults to `192.168.50.0/24`, not the old lab
`192.168.1.0/24`.

### 5.2a Address plan (decided 2026-09-22)

The RT-BE58U keeps its default LAN IP `192.168.50.1`. The repo was migrated to
match — only the third octet changed, so the old numbering conventions still hold:

| Range | Use |
|---|---|
| `192.168.50.1` | router / gateway |
| `192.168.50.10` | host `Ghali-T1700`, container registry |
| `192.168.50.101`–`.106` | Raspberry Pi companion computers (`robotN`) |
| `192.168.50.151`–`.156` | RoboMaster units, `RMn` → `.(150+n)` |

Files already updated to this plan:
`ros2_swarm/src/swarm_bringup/config/swarm_config.yaml`,
`ros2_swarm/src/swarm_bringup/launch/launch_centralized_swarm.py`,
`swarm_ansible_deploy/inventory.ini`.

`docs/resources/netgear_fixed_IP_DHCP.md` is **not** updated — it documents the
retired WNDR3700v4 and is kept as history.

Reservations can only be entered once a robot has associated at least once, since
the router needs its MAC. Known so far: RM4 `60:60:1f:ce:27:d9`; the S1
`48:1c:b9:62:29:c7`. Work one robot at a time — provision (§5.3), find it in
**Network Map → Clients**, add the reservation, reboot the robot, then pin its IP
in the config.

### 5.3 Provision one robot onto the new access point

Credentials can only be set through the RoboMaster **app**:

1. Robot's connection-mode switch → **direct**, then **power-cycle**
   (the switch is NOT hot-swappable)
2. Phone joins the robot's hotspot `RMS1-xxxxxx` / `RMEP-xxxxxx`, PSK **`12341234`**
3. App → connection settings → **networking / router mode** → enter the new SSID and password
4. Robot reboots into networking mode and joins the new access point

**Gotcha:** a robot given bad credentials strands itself — it broadcasts **no** hotspot
and joins **no** network. Recover with mode switch → direct + power-cycle.

**Gotcha:** an iPhone cannot host its hotspot and join the robot's hotspot at the same
time, so provisioning against a phone hotspot has to be interleaved.

### 5.4 Confirm the robot is on the network

```bash
# Sweep the subnet
for i in $(seq 1 254); do (ping -c1 -W1 -I eno1 192.168.50.$i >/dev/null 2>&1 &) ; done
sleep 8
ip neigh show dev eno1 | grep -v FAILED
```

DJI OUIs: `60:60:1F` and `48:1C:B9`. Known lab devices: `.1` router (the
RT-BE58U), and `20:0b:74:86:f7:ae` (unrelated, wandered between `.3`/`.4` on the
old subnet).

If the sweep finds nothing, check the access point's **Attached Devices** page — a robot can be
associated yet ARP-unreachable. That distinction is exactly what identified this fault.

### 5.5 Pass/fail test 1 — link quality

```bash
ping -c 40 -i 0.3 -W 2 -I eno1 <gateway_ip>   # baseline, expect 0%
ping -c 40 -i 0.3 -W 2 -I eno1 <robot_ip>     # expect <10%
```

**PASS:** robot loss under ~10%, **minimum RTT in single-digit milliseconds**.
**FAIL:** loss above ~20%, or a minimum-RTT floor of tens of milliseconds.

The min-RTT floor is the more reliable indicator than the loss percentage — it was
the signature that survived every environmental change.

### 5.6 Pass/fail test 2 — the driver

```bash
source /opt/ros/humble/setup.bash
source ~/projects/ros2_humble_swarm/ros2_robomaster/install/setup.bash
RM_DIAGNOSTICS=verbose ros2 launch robomaster_ros s1.launch \
    conn_type:=sta robot_ip:=<robot_ip>
```

Note `launch_centralized_swarm.py` does **not** expose `conn_type` (it takes
`main.launch`'s `sta` default), so use `s1.launch` directly for single-robot tests.
`RM_DIAGNOSTICS=verbose` forces SDK logging to DEBUG and overrides
`lib_log_level=ERROR` — no file edit needed.

**PASS looks like:**
```
Try to connect via sta to robot with sn ... (1/5)
Connected                                        <- within a few hundred ms
Enabled modules: Battery, Camera, Chassis, LED, Speaker, Gimbal, Blaster
link      : robot=...  arp=PRESENT  rssi=-33dBm  ssid=...
heartbeat : n=10 mean=1.00s max=1.04s
```
Run at least 60 s. There must be **no** `Disconnected`, **no** `RECV TimeOut`, and
**no** reconnect attempts. Heartbeat mean should sit at 1.00 s; occasional 2 s gaps
are tolerable (the driver's timeout is 5 s).

**FAIL looks like:** repeated `Connection attempt N/5 failed`, or `Connected` followed
by `Disconnected` within seconds.

### 5.7 Scale up

Only after one robot passes both tests: provision the remaining robots, confirm the
**DHCP address reservations** from §5.2 cover each of them, update `robot_ips` in
`ros2_swarm/src/swarm_bringup/launch/launch_centralized_swarm.py:36` (IPs are
hard-coded there — a changed lease will break the launch), then run the full
`launch_centralized_swarm.py`.

---

## 6. Reference — robot inventory

| Robot | Serial number | Notes |
|---|---|---|
| RM1 | 159CKC50070ECX | |
| RM2 | 159CKC50070E5N | |
| RM3 | 159CG9J0050797 | |
| RM4 | 159CG9V0050HED | MAC `60:60:1f:ce:27:d9`, was `192.168.1.154` on the old subnet |
| RM5 | 159CKCH0070F8S | |
| RM6 | 159CKC50070EVP | |

The unit with MAC `48:1c:b9:62:29:c7` broadcasts `RMS1-6229c7` and is an **S1**, not an EP.

## 7. Diagnostic logs

Raw evidence from this session:

- `/tmp/driver_test.log` — networking mode via NETGEAR48, 10/10 handshake failures
- `/tmp/driver_ap.log` — direct-connect mode, 73 s clean
- `/tmp/driver_hotspot.log` — networking mode via phone hotspot, 73 s clean

Copy these somewhere permanent if they are still needed; `/tmp` does not survive a reboot.
