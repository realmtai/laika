# Laika (Unitree Go2) — Host Reconnaissance Report

> **Scope:** Live investigation of the two Go2 addresses on the robot's Wi-Fi network. <br>
> **Date:** 2026-07-09 <br>
> **Observer:** workstation at `192.168.123.230` (DHCP on `TMU-LAIKA-2.4G`, interface `en0`) <br>
> **Network:** `192.168.123.0/24`, gateway `192.168.123.1` (GL-BE3600 router mounted on the robot)
>
> **Credentials note:** SSH login values were taken from this wiki ([2.2 Connecting Via Ethernet](2.2-Connecting-Via-Ethernet), [3.4 Go2 Unitree ROS2](3.4-Go2-Unitree-ROS2)). Per wiki policy ([A.1 Template](A.1-Template)), no username or password values are recorded in this file.

---

## Summary

| Address | Role (per wiki) | Reachability | Access | Headline finding |
| :--- | :--- | :--- | :--- | :--- |
| `192.168.123.18` | Go2 onboard computer | Ping + SSH (port 22) | SSH with wiki-documented credentials | NVIDIA **Orin NX** dev kit, Ubuntu 20.04, ROS **Foxy + Noetic**, RealSense D435i, Docker + NoMachine running |
| `192.168.123.161` | Go2 robot body (ping target via router) | Ping only | **No SSH** (port 22 refused); no credentials exist in the wiki | Firmware services on ports **80** and **10001**; both silent to all HTTP probes |

---

## 1. `192.168.123.18` — Onboard Computer (Jetson Orin NX)

### Access

- SSH service on port 22; authenticates with the wiki-documented login (see credentials note above).
- Verified session: remote `hostname` → `ubuntu`.

### Hardware specification

| Component | Value |
| :--- | :--- |
| SoC / board | **NVIDIA Orin NX Developer Kit** (`/proc/device-tree/model`) |
| JetPack / L4T | **R35.1** (L4T release, GCID 32827747, `t186ref`, built 2023-03-19) |
| CPU | 8× ARMv8 (Cortex-A78/A55-class), 1 socket, max `1984 MHz`, min `115.2 MHz`. **Only cores 0–3 online; 4–7 offline** (see interesting notes) |
| RAM | **15 GiB** (+7.5 GiB swap) — the 16 GB Orin NX variant |
| Storage | NVMe `/dev/nvme0n1p1`, **469 GB** total, 30 GB used (7%) |
| GPU driver | `nvidia-smi` not available (integrated Orin GPU; no discrete driver query) |
| USB | **Intel RealSense D435i** (`8086:0b3a`) — the only external USB device |
| Video devices | `/dev/video0` … `/dev/video5` |
| Serial/UART | `/dev/ttyTHS0`, `ttyTHS3`, `ttyTHS4` (Tegra high-speed UARTs) |
| NIC (eth0) | `192.168.123.18/24`, MAC **`4c:bb:47:ab:f4:2e`**, default route via `192.168.123.1` |
| Other interfaces | `docker0` (DOWN, `172.17.0.1/16`), `dummy0` (DOWN), `lo` |
| Uptime at capture | 18 min, load average `0.11, 0.03, 0.01` |

### Operating system

| Item | Value |
| :--- | :--- |
| OS | **Ubuntu 20.04.5 LTS (Focal Fossa)**, aarch64 |
| Kernel | `5.10.104-tegra` (PREEMPT, 2023-03-19) — matches L4T R35.1 |
| Desktop | GNOME (`gnome-shell` under `gdm`), Xorg on `tty1` |

### Installed software and packages

| Item | Value |
| :--- | :--- |
| dpkg packages | **3322** installed |
| pip3 (python3) packages | **203** |
| ROS 2 | **Foxy** (`/opt/ros/foxy`) — full desktop install (ament, actionlib, action-msgs, …) |
| ROS 1 | **Noetic** (`/opt/ros/noetic`) |
| Python ROS tooling | `colcon-ros 0.4.1`, `rosdep 0.22.2`, `rospkg 1.5.0` |
| Vision | `opencv-python 4.13.0.92` |
| NVIDIA/Jetson | `nvidia-l4t-jetson-io`, `nvidia-l4t-jetson-multimedia-api`, `nvidia-l4t-jetsonpower-gui-tools`, `python3-jetson-gpio` |
| Container runtime | **Docker** + containerd (`/opt/containerd`, `/opt/nvidia`); `dockerd` running as root |
| Remote access | **NoMachine** (`/usr/NX/bin/nxserver.bin` running; `~/nomachine.sh` present) |
| Notable stock service | `unitree-upgrade.service` — **active (running)** |

### Home directory and workspaces

| Path | Size | Notes |
| :--- | :--- | :--- |
| `~/unitree` | 604 MB | Contains `Odometer_service/` (Unitree-supplied odometer stack, dated Nov 2023) and `usb3.0激活.docx` ("USB 3.0 activation" document, in Chinese) |
| `~/cyclonedds_ws` | 95 MB | colcon workspace: `build/`, `install/`, `src/`, plus a standalone `cyclonedds.xml` (the DDS interface config used per [3.4 Go2 Unitree ROS2](3.4-Go2-Unitree-ROS2)) |
| `~/unitree_ws` | 8 KB | colcon workspace with an **empty** `src/` — `unitree_ros2` is not currently checked out here |

### Running processes of note

- `dockerd` + `containerd` (root) — container stack live, but the `unitree` user has no permission to talk to the Docker socket.
- `nxserver.bin` / `nxnode.bin` — NoMachine remote-desktop daemon.
- `gnome-shell`, `Xorg` — full desktop session running on the robot.
- No ROS 2 nodes, no `sport_mode`, no camera or LiDAR driver processes were running at capture time (robot freshly booted, 18 min uptime).

### Interesting observations

1. **Half the CPU is parked:** 4 of 8 Orin NX cores are offline (`On-line: 0-3`). This is a visible power/perf cap; worth confirming whether it is stock Unitree behaviour or a tuned setting.
2. **Two ROS generations side by side:** both Noetic (ROS 1) and Foxy (ROS 2) are installed — Foxy is the one the wiki targets for `unitree_ros2`.
3. **`unitree_ws` is empty:** the documented `unitree_ros2` checkout ([3.4](3.4-Go2-Unitree-ROS2)) is not present at capture time, so the DDS bridge is not currently built on the robot.
4. **Docker exists but is locked to root** — the RealSense work in [5.3](5.3-Configuring-the-Intel-RealSense-D435i-Depth-Camera) will need `sudo docker` or socket permissions for the `unitree` user.
5. **NoMachine is installed and running** — a second remote-access path besides SSH (another reason to keep credentials off this wiki).
6. **`Odometer_service` (604 MB)** in `~/unitree` is Unitree's own odometry service, preinstalled with the stock image.
7. **Only one sensor on USB:** the RealSense D435i. No L1 LiDAR appeared on USB (per [5.4](5.4-FAST-LIO-on-the-Go2) it may attach elsewhere), and no LiDAR driver process was running.

---

## 2. `192.168.123.161` — Robot Body Controller

### Access attempts

- SSH (port 22): **connection refused** — the host is up and answering, but it does not run an SSH daemon.
- No credentials for this address exist anywhere in the wiki; the documented login applies only to the onboard computer at `192.168.123.18`. Per the wiki, robot-body details are fixed by Unitree firmware.

### External probing (from `192.168.123.230`)

| Probe | Result |
| :--- | :--- |
| ICMP ping | Replies, ~4–22 ms RTT, 0% loss |
| Port sweep (1–1024 + ~40 common/high ports, 1 s timeouts) | **Open: `80/tcp`, `10001/tcp`** — everything else closed or filtered |
| HTTP `GET /` on `:80` (HTTP/1.1 and 1.0) | TCP connect succeeds, server sends **no response** and closes (`Empty reply from server`) |
| HTTP `POST` and common paths (`/index.html`, `/ota`, `/upgrade`, `/api`, `/status`, `/robot`, `/device`, `/info`) | Same: no response |
| TLS on `:443` | Closed |
| Raw banner read on `:80` and `:10001` (8 s wait) | Silence — no protocol banner |
| `GET / HTTP/1.1` sent to `:10001` | Silence |
| MAC address | **Not retrievable:** macOS kept no visible ARP table entries (`arp -an` empty) despite direct L2 reachability. (For reference, the onboard computer's `eth0` MAC is `4c:bb:47:ab:f4:2e`.) |

### Interpretation

- `.161` is the **Go2 body's own network endpoint** (the wiki's "ping target via router"), distinct from the Jetson onboard computer at `.18`.
- The two listening ports behave like **firmware services using a non-HTTP protocol**: they accept TCP but answer nothing to HTTP GET/POST or raw banner reads. Port `80` is consistent with the stock Unitree web/OTA service exposed by the body, and `10001` is a secondary (likely internal telemetry/control) port. Identifying the exact protocol would require Unitree firmware documentation or packet capture while operating the robot.
- There is **no shell access path to this address** from the wiki; interaction is via the Unitree GO app, the onboard computer, or Unitree's SDK/proprietary services.

---

## Method and caveats

- SSH commands were run through the `ssh-to-host` skill (`run` mode); all values in section 1 were read live from the robot on 2026-07-09.
- Port scans used parallel `nc -z` probes (no `nmap` on the observer): full 1–1024 sweep plus a sample of common and high ports with 1–2 s timeouts. An obscure high port could theoretically be missed.
- The robot had been up ~18 minutes at capture; transient state (running nodes, container list) may differ in normal operation.
- No password or username values appear in this document, per wiki policy; see the credentials note at the top.

### Related wiki pages

- [2.2 Connecting Via Ethernet](2.2-Connecting-Via-Ethernet) — SSH access to `.18`
- [2.3 Connecting Via Router](2.3-Connecting-Via-Router) — the `192.168.123.x` topology
- [3.4 Go2 Unitree ROS2](3.4-Go2-Unitree-ROS2) — `unitree_ros2` / CycloneDDS setup
- [5.3 RealSense D435i](5.3-Configuring-the-Intel-RealSense-D435i-Depth-Camera) / [5.4 FAST-LIO](5.4-FAST-LIO-on-the-Go2) — sensors found/expected on the robot
