# Raspberry Pi GPS‑PPS Stratum‑1 Time Server (Chrony) + OPNsense & TrueNAS Clients

> A reproducible, end‑to‑end guide to build a home/SMB stratum‑1 time server with a Raspberry Pi and GPS‑PPS, and use it as the authoritative NTP source for OPNsense and TrueNAS SCALE.

This repo captures the exact setup that produced stable **nanosecond‑level** PPS offsets and clean synchronization across the network.

* Hostnames used in examples:

  * **gps** > Raspberry Pi 3B running *chrony* + *gpsd* (Ethernet static IP **192.168.2.2**; Wi‑Fi disabled)
  * **opnsense** > firewall/router running *ntpd* (client of the Pi)
  * **truenas** > TrueNAS SCALE (client of the Pi, chrony backend)

---

## Contents

* [Hardware](#hardware)
* [Wiring & Environment](#wiring--environment)
* [Raspberry Pi OS Setup](#raspberry-pi-os-setup)
* [GPSD & PPS](#gpsd--pps)
* [Chrony configuration (Pi)](#chrony-configuration-pi)
* [Network & Security](#network--security)
* [Disable Wi‑Fi on the Pi](#disable-wi-fi-on-the-pi)
* [OPNsense as an NTP client](#opnsense-as-an-ntp-client)
* [TrueNAS SCALE as an NTP client](#truenas-scale-as-an-ntp-client)
* [Validation](#validation)
* [Troubleshooting](#troubleshooting)
* [Data collection scripts (CSV + plots)](#data-collection-scripts-csv--plots)
* [Why NTS?](#why-nts)

---

## Hardware

* Raspberry Pi **3B** (Pi 4/5 also fine)
* microSD card (≥ 8GB)
* **GPS receiver** (“GPS puck”) capable of NMEA sentences; **PPS** output **recommended**

  * Option A (common): USB GPS puck that exposes NMEA (/dev/ttyACM0 or /dev/ttyUSB0). Some pucks also expose PPS via kernel line discipline; others **do not**.
  * Option B (robust): GNSS HAT or module wired to Pi’s GPIO (NMEA on UART + **PPS on a GPIO pin**).
* Outdoor‑friendly GPS antenna placement or puck with clear sky view (window sill often OK)
* Ethernet to your LAN switch (low jitter; do **not** rely on Wi‑Fi)

> **Tip:** If your puck does **not** provide PPS via USB (many don’t), use a GNSS HAT/module and wire PPS to a GPIO pin (e.g., GPIO 18 / pin 12). PPS is what enables sub‑microsecond performance.

---

## Wiring & Environment

### If using **GPIO PPS** (recommended)

* Power GNSS module from 3.3V (pin 1) and GND (pin 6)
* GNSS TX → Pi RX (GPIO 15 / pin 10)
* GNSS RX → Pi TX (GPIO 14 / pin 8) (optional if you only need one‑way NMEA)
* **PPS → GPIO 18 (pin 12)** (default mapping used below)

> **Voltage caution:** Ensure the GNSS module’s logic levels are **3.3V** compatible before wiring to the Pi.

### Physical placement

* Place the puck/module close to a window or outdoors
* Avoid long USB cables with questionable shielding
* Keep away from noisy PSUs, switching supplies, and big RF emitters

---

## Raspberry Pi OS Setup

Assumes Raspberry Pi OS (Debian‑based). Update and install dependencies:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y chrony gpsd gpsd-tools pps-tools gawk gnuplot
```

Set a **static DHCP reservation** for the Pi’s MAC in your router/DHCP server (recommended), or set a static IP on the Pi if you prefer.

Set the timezone to UTC (recommended for time servers):

```bash
sudo timedatectl set-timezone UTC
```

---

## GPSD & PPS

### 1) Enable UART / PPS overlays (if using GPIO PPS)

Edit `/boot/firmware/config.txt` (on older images: `/boot/config.txt`) and append:

```ini
# Free the PL011 UART on Pi 3 (if using on-board UART for NMEA)
dtoverlay=pi3-disable-bt
enable_uart=1

# PPS on GPIO 18 (pin 12) — change gpiopin if you wired differently
dtoverlay=pps-gpio,gpiopin=18
```

Reboot to apply overlays:

```bash
sudo reboot
```

After reboot, verify PPS device exists:

```bash
ls -l /dev/pps*
sudo dmesg | grep -i pps
```

You should see `/dev/pps0` and kernel messages about PPS.

### 2) Configure gpsd

Edit `/etc/default/gpsd`:

```bash
sudo nano /etc/default/gpsd
```

Set values (adjust device path to your GPS):

```ini
START_DAEMON="true"
USBAUTO="true"
DEVICES="/dev/ttyACM0"   # or /dev/ttyUSB0 or /dev/ttyAMA0 for UART
GPSD_OPTIONS="-n"         # push data to clients without waiting
```

Enable and start services:

```bash
sudo systemctl enable --now gpsd
sudo systemctl status gpsd --no-pager
```

Quick sanity checks:

```bash
# Text mode monitor
cgps -s
# or
gpsmon
```

You should see satellites and time. It may take several minutes for first fix.

> **Using PPS via USB?** On some USB serial adapters, you must attach the PPS line discipline to the serial port:
>
> ```bash
> sudo apt install -y setserial
> sudo ldattach 18 /dev/ttyUSB0   # replace with your actual serial device
> ```
>
> Check for `/dev/pps0` again after attaching.

---

## Chrony configuration (Pi)

Edit `/etc/chrony/chrony.conf` to use GPSD (NMEA) + PPS and serve your LAN. Below is a **known‑good** baseline:

```conf
# =====================
# Chrony – gps + pps
# =====================

# Use NMEA from gpsd via shared memory (SHM 0)
refclock SHM 0 offset 0.5 delay 0.2 refid NMEA

# Use kernel PPS device and lock it to NMEA
refclock PPS /dev/pps0 refid PPS lock NMEA prefer

# Step the clock on boot if the offset is large
makestep 1.0 3

# Allow kernel to sync RTC on clean shutdown
rtcsync

# Drift file
driftfile /var/lib/chrony/chrony.drift

# Log files (optional)
logdir /var/log/chrony

# Serve NTP to your LAN (replace with your subnet)
allow 192.168.2.0/24

# Command socket
bindcmdaddress 127.0.0.1
cmdport 323
cmdallow 127.0.0.1

# =====================
# Optional: NTS upstream (authenticated NTP)
# Some networks filter pool NTP; using NTS with well-known providers can help.
# NTS-KE over TCP/4460; timing still uses UDP/123.
# =====================
server time.cloudflare.com iburst nts
server nts.netnod.se iburst nts
```

Restart chrony and gpsd:

```bash
sudo systemctl restart gpsd chrony
sudo systemctl enable chrony
```

Validate on the Pi:

```bash
# Sources (expect PPS selected `#*` and NMEA `#x`/`#?`)
sudo chronyc sources -v

# Tracking (Stratum 1, refid PPS, tiny offsets/skew)
sudo chronyc tracking

# Auth status of NTS upstream (optional)
sudo chronyc -N authdata
```

Sample healthy output (PPS in control):

```
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
#x NMEA                          0   4   377     8   +126ms[ +126ms] +/-  102ms
#* PPS                           0   4   377    10   +124ns[ +213ns] +/- 5256us
^- nts.netnod.se                 1  10   377    76   -591us[ -591us] +/-   24ms
```

> **Rule of thumb:** NMEA has tens–hundreds of ms noise. PPS should be **selected** and show **µs→ns** offsets.

---

## Network & Security

Open the Pi’s firewall (UFW example) to allow NTP from your LAN:

```bash
sudo apt install -y ufw
sudo ufw allow proto udp from 192.168.2.0/24 to any port 123
sudo ufw enable
sudo ufw status
```

**SSH hardening (key‑based):**

```bash
# On your admin workstation
ssh-keygen -t ed25519 -C "pi-time-admin"
ssh-copy-id pi@192.168.2.2

# On the Pi
sudo nano /etc/ssh/sshd_config
# Ensure:
#   PubkeyAuthentication yes
#   PasswordAuthentication no
#   PermitRootLogin prohibit-password (or no)

sudo systemctl reload ssh
```

---

## Disable Wi‑Fi on the Pi

For a time server, use **wired Ethernet** only.

**Option A – Device tree overlay:** add to `/boot/firmware/config.txt`:

```ini
dtoverlay=disable-wifi
```

**Option B – Systemd:**

```bash
sudo rfkill block wifi
sudo systemctl disable --now wpa_supplicant
```

Reboot and verify `wlan0` is down.

---

## OPNsense as an NTP client

OPNsense uses **ntpd** (not chrony). It will happily sync from your Pi.

1. **System → Settings → General → Network Time** *(or)* **Services → Network Time**
2. **General** tab:

   * **Time servers**: add your Pi’s **Ethernet IP** (e.g., `192.168.2.2`)
   * Tick **IBurst**
   * Optionally tick **Prefer**
   * **Interfaces**: bind to LAN / required interfaces
   * Leave access‑restriction defaults unless you know you need custom rules
3. Save & Apply.

Verify on OPNsense shell:

```sh
sockstat -4 -l | grep ':123'
ntpq -pn
cat /var/etc/ntpd.conf
```

Expected `ntpq -pn` (Pi as upstream):

```
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*192.168.2.2    .PPS.            1 u   56   64  377    1.156   +0.047   0.031
```

> **Note:** If you do not have a physical PPS wired into the firewall box, **do not** enable local PPS/refclock on OPNsense. Just point it at the Pi.

---

## TrueNAS SCALE as an NTP client

Newer TrueNAS SCALE uses **chrony** under the hood.

* **System Settings → General → NTP Servers → Add**

  * **Address**: `192.168.2.2`
  * Tick **IBurst** and **Prefer**
  * Leave Min/Max Poll defaults (6/10) unless you have a reason to change
* Save, then check via shell:

```bash
systemctl status chrony --no-pager
chronyc sources -v
chronyc tracking
```

If you want TrueNAS to consider one source sufficient, set in `/etc/chrony/chrony.conf` (persist carefully):

```
minsources 1
```

> GUI updates may regenerate config. Prefer GUI when possible.

---

## Validation

On the **Pi**:

```bash
chronyc tracking
chronyc sources -v
```

Look for:

* `Stratum : 1` and `Reference ID : PPS`
* Offsets in tens of ns to low µs
* Stable `Skew` ≪ 1 ppm after some hours

From a **client** (e.g., TrueNAS):

```bash
chronyc -N -n ntpdata 192.168.2.2
```

Check that the server advertises `Stratum : 1`, normal leap status, and small offset.

From **any Linux client**:

```bash
sudo tcpdump -ni any 'udp port 123 and host 192.168.2.2' -vv -c 4
```

You should see clean client/server exchanges.

---

## Troubleshooting

* **No `/dev/pps0`**

  * If using GPIO: confirm `dtoverlay=pps-gpio,gpiopin=18` and reboot
  * If using USB: the device may not expose PPS; try a GNSS HAT with GPIO PPS
  * Try `sudo ldattach 18 /dev/ttyACM0` (replace device) and re‑check

* **`chronyc sources` never selects PPS**

  * Ensure `refclock PPS /dev/pps0 lock NMEA prefer` **and** `refclock SHM 0 ... refid NMEA`
  * Let it run for 10–20 minutes to stabilize

* **Clients can’t query the Pi (no response on UDP/123)**

  * `ufw status` → ensure LAN is allowed to 123/udp
  * `ss -lunp | grep ':123 '` on the Pi → ensure chronyd is listening on 0.0.0.0:123

* **ISP/network filters NTP (UDP 123)**

  * NTS uses TCP/4460 for key exchange **but still uses UDP/123** for timing
  * If pools are blocked but specific NTS servers are allowed, configure `server ... nts` lines to permitted providers
  * Worst case: rely solely on GPS‑PPS (works fine for LAN clients; just no external cross‑check)

* **OPNsense `restrict` syntax errors**

  * ntpd **does not** accept CIDR on `restrict` lines. Use mask form:

    * ✅ `restrict 192.168.2.0 mask 255.255.255.0 nomodify notrap`
    * ❌ `restrict 192.168.2.0/24 ...`

* **High jitter on Wi‑Fi**

  * Don’t. Use Ethernet. Disable Wi‑Fi on the Pi.

---

## Data collection scripts (CSV + plots)

This repo includes a simple set of scripts to log chrony metrics and plot stability over time.

### `scripts/collect.sh`

```bash
#!/usr/bin/env bash
# Collect chrony stats into CSV files under ./data
# Run via cron (e.g., every minute) or a systemd timer.
set -euo pipefail

DIR="$(dirname "$0")/.."
DATA="$DIR/data"
mkdir -p "$DATA"
TS="$(date -u +%Y-%m-%dT%H:%M:%SZ)"

# tracking.csv: timestamp, last_offset, rms_offset, freq, skew, root_delay, root_dispersion
chronyc tracking | awk -v ts="$TS" '
  BEGIN{lo=ro=f=freq=sk=rd=rdp=""}
  /Last offset/{lo=$3}
  /RMS offset/{ro=$3}
  /Frequency/{freq=$3}
  /Skew/{sk=$3}
  /Root delay/{rd=$4}
  /Root dispersion/{rdp=$4}
  END{print ts "," lo "," ro "," freq "," sk "," rd "," rdp}
' >> "$DATA/tracking.csv"

# sourcestats.csv: timestamp + each source line
chronyc sourcestats -v | awk -v ts="$TS" 'BEGIN{p=0}
  /^Name/ {p=1; next}
  p && NF>0 {gsub(/\r/, ""); print ts "," $0}
' >> "$DATA/sourcestats.csv"

# sources.csv: timestamp + raw sources table (useful for reach/poll)
chronyc sources -v | awk -v ts="$TS" 'BEGIN{p=0}
  /^MS / {p=1; next}
  p && NF>0 {gsub(/\r/, ""); print ts "," $0}
' >> "$DATA/sources.csv"
```

### `scripts/plot_offset.gp` (gnuplot)

```gnuplot
set datafile separator ","
set xdata time
set timefmt "%Y-%m-%dT%H:%M:%SZ"
set format x "%H:%M\n%m-%d"
set grid
set title "Chrony Last Offset"
set xlabel "UTC time"
set ylabel "seconds"
plot "data/tracking.csv" using 1:2 with lines title "last_offset"
```

Run:

```bash
mkdir -p data
./scripts/collect.sh   # run once
# (add to cron: */1 * * * * /path/to/scripts/collect.sh)

gnuplot -persist scripts/plot_offset.gp
```

---

## Why NTS?

Some networks and ISPs throttle or filter traditional NTP queries to public pools. **NTS (Network Time Security)** adds TLS‑based key exchange (TCP/4460) and authenticated NTP and is increasingly preferred by providers like Cloudflare and Netnod.

* In practice, NTS gives you: integrity/authentication of the NTP exchange and sometimes more reliable upstream access.
* Your LAN clients still talk plain NTP to the Pi (on your LAN), but the Pi can use NTS to its upstreams for assurance.

> Reminder: Even with NTS, the time packets themselves still use UDP/123 after the initial TLS key exchange.

---

## Example outputs (healthy system)

**Pi (server):**

```
Reference ID    : 50505300 (PPS)
Stratum         : 1
Ref time (UTC)  : 2025-08-20 16:24:12
System time     : 0.000000044 seconds slow of NTP time
Last offset     : -0.000000018 seconds
RMS offset      : 0.000000638 seconds
Frequency       : 0.040 ppm slow
Skew            : 0.001 ppm
```

**OPNsense (client):**

```
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*192.168.2.2    .PPS.            1 u   56   64  377    1.156   +0.047   0.031
```

---

## License

MIT

---

## Credits

Thanks to the open‑source communities behind **chrony**, **gpsd**, and **OPNsense/FreeBSD**, and to NTS providers like **Cloudflare** and **Netnod** for reliable public services.
