Raspberry Pi Stratum-1 NTP Server (GPS/PPS) + OPNsense + TrueNAS SCALE
Turn a Raspberry Pi + GPS puck into a stratum-1 time source using chrony with GPS (NMEA) and PPS. Then point OPNsense and TrueNAS SCALE at it for rock-solid time across your network.
✅ This README is written for Pi OS/Debian-like systems and recent OPNsense/TrueNAS SCALE. Commands assume a Pi on Ethernet at 192.168.3.92 (Wi-Fi disabled) and a LAN 192.168.3.0/24. Adjust IPs to taste.
Diagram
[GPS puck] --(serial NMEA + PPS)--> [Raspberry Pi 3B/4]
                                         |
                                         |  NTP server (UDP/123), stratum 1
                                         v
                           +-------------+-------------+
                           |                           |
                      [OPNsense]                  [TrueNAS SCALE]
                 NTP client/server               NTP client (chrony)
Hardware
Raspberry Pi 3B/4 (with reliable power, Ethernet)
GPS puck that provides NMEA (serial/USB) and PPS
If using a HAT or GPIO PPS: use pps-gpio overlay (commonly GPIO18).
If using USB GPS with PPS: device may expose /dev/ttyACM* and /dev/pps0.
Optional: short SMA/coax leads, sky view for GPS antenna (window ledge works; roof is best).
Network plan
Reserve a static DHCP lease for the Pi (e.g. 192.168.3.92).
Disable Wi-Fi on the Pi (recommended) or bind chrony to the Ethernet IP only.
Open UDP/123 inside your LAN (no internet exposure).
Raspberry Pi setup
1) OS packages
sudo apt update
sudo apt install -y chrony gpsd gpsd-clients pps-tools
2) Enable PPS
Pick one path (GPIO HAT) or (USB PPS):
A) GPIO PPS (common for timing HATs)
# Add overlay and disable Bluetooth serial if your HAT uses ttyAMA0
echo 'dtoverlay=pps-gpio,gpiopin=18' | sudo tee -a /boot/firmware/config.txt
# (Optional) If your GPS uses the primary UART (ttyAMA0), also:
# echo 'dtoverlay=miniuart-bt' | sudo tee -a /boot/firmware/config.txt
sudo reboot
B) USB PPS
Plug in the GPS. You should see /dev/ttyACM0 (or /dev/ttyUSB0) and /dev/pps0.
Verify PPS:
ls -l /dev/pps* /dev/ttyACM* /dev/ttyUSB* || true
sudo ppstest /dev/pps0   # should print timestamps (Ctrl+C to stop)
3) Configure gpsd
Tell gpsd which device carries NMEA (serial stream). For GPIO HATs: often /dev/ttyAMA0. For USB: typically /dev/ttyACM0 or /dev/ttyUSB0.
sudo sed -i 's|^DEVICES=.*|DEVICES="/dev/ttyAMA0 /dev/ttyACM0 /dev/ttyUSB0"|' /etc/default/gpsd
sudo sed -i 's|^START_DAEMON=.*|START_DAEMON="true"|' /etc/default/gpsd
sudo systemctl enable --now gpsd
Quick check:
cgps -s   # or: gpsmon
# You should see satellites/lock after a few minutes with sky view
4) Chrony configuration (stratum-1)
Backup and write a clean config:
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak.$(date +%s)

sudo tee /etc/chrony/chrony.conf >/dev/null <<'EOF'
# ===== Chrony (Raspberry Pi) =====
# Make single-source operation ok (we're a GPS/PPS time source)
minsources 1

# Step the clock early at boot if needed
makestep 1.0 3
rtcsync
driftfile /var/lib/chrony/chrony.drift
logdir /var/log/chrony

# --- GPS via gpsd (NMEA) ---
# Shared memory from gpsd. SHM 0 is the standard NMEA channel.
# We'll calibrate 'offset' later once stable.
refclock SHM 0 offset 0.5 delay 0.2 refid NMEA

# --- PPS from kernel ---
# Lock PPS to NMEA and prefer it as the disciplining source
refclock PPS /dev/pps0 refid PPS lock NMEA prefer

# --- (Optional) External sanity check via NTS ---
# These do not have to succeed once PPS is solid, but can help at boot.
# server time.cloudflare.com iburst nts
# server nts.netnod.se      iburst nts

# Serve time to LAN
allow 192.168.3.0/24

# Listen only on the Ethernet IP (avoid Wi-Fi / wildcard)
bindaddress 192.168.3.92

# Chrony command socket (local admin)
bindcmdaddress 127.0.0.1
cmdport 323
cmdallow 127.0.0.1
EOF

sudo systemctl enable --now chrony
(If you want to hard-disable Wi-Fi entirely:)
echo 'dtoverlay=disable-wifi' | sudo tee -a /boot/firmware/config.txt
sudo reboot
5) Open the firewall (LAN only)
If you use UFW:
sudo ufw allow proto udp from 192.168.3.0/24 to any port 123
sudo ufw status
6) Verify chrony on the Pi
Give GPS a few minutes to lock (PPS will go “selected” fast once lock is valid):
chronyc sources -v
chronyc tracking
Healthy example (once stable):
MS Name/IP address   Stratum Poll Reach LastRx Last sample
#x NMEA                    0    4   377      8   -300ms[ -300ms] +/- 100ms
#* PPS                     0    4   377      9    -80ns[ -120ns] +/- 0.9ms

Reference ID    : 50505300 (PPS)
Stratum         : 1
...
Leap status     : Normal
Calibrate NMEA offset (optional, after a day or two):
chronyc sourcestats -v → take NMEA mean offset (e.g. -0.350 s) and set the opposite sign in the NMEA refclock line:
refclock SHM 0 offset 0.350 delay 0.2 refid NMEA
sudo systemctl restart chrony
OPNsense (as NTP client/server)
OPNsense uses ntpd. Easiest, point it at the Pi.
GUI path
Services → Network Time → General
Time servers: 192.168.3.92 (✓ iburst)
(Optional) Interfaces: select LAN (and loopback) so it listens there only.
(Optional) enable Syslog logging and NTP graphs.
Click Save and Restart the Network Time service.
Note on access rules: ntpd uses restrict lines which do not accept CIDR. If you add custom restrictions, use the mask form:
restrict 192.168.3.0 mask 255.255.255.0 nomodify notrap nopeer
(CIDR like 192.168.3.0/24 will trigger “unusable” syntax errors.)
Verify OPNsense
ntpq -pn
Example when synced to the Pi (stratum 1):
remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*192.168.3.92    .PPS.        1 u   56   64  377    1.156   +0.047   0.031
If you also added custom PPS on OPNsense (not required if the Pi is your stratum-1), you’ll see a 127.127.22.0 refclock. Most setups only need the Pi.
TrueNAS SCALE (chrony client)
TrueNAS SCALE uses chrony.
GUI steps
System Settings → General → NTP Servers → Add
Address: 192.168.3.92
Enable IBurst and optionally Prefer
Save, then Start/Enable the chrony service if not running
TrueNAS may also keep its default public servers. That’s fine, but if you want it to be satisfied with just your Pi, set minsources 1 in chrony. (Edits may be reverted by updates; use with caution.)
(Optional) Allow single source in TrueNAS
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak.$(date +%s)
echo 'minsources 1' | sudo tee -a /etc/chrony/chrony.conf
sudo systemctl restart chrony
Verify TrueNAS
chronyc sources -v
chronyc tracking
You want to see the Pi selected:
MS Name/IP address   Stratum Poll Reach LastRx Last sample
^* 192.168.3.92            1   6   377    100    -2ms[   0ms] +/- 20ms
Packet peek (on TrueNAS):
sudo tcpdump -ni any 'udp port 123 and host 192.168.3.92' -vv -c 6
Diagnostics & Useful Commands
On the Pi
# Who is selected?
chronyc sources -v
chronyc tracking

# PPS alive?
sudo ppstest /dev/pps0

# NTS status (if enabled)
chronyc -N authdata

# Ports open?
ss -lunp | grep ':123 '
On OPNsense
sockstat -4 -l | grep ':123'     # listeners
ntpq -pn                          # peers
ntpq -crv                         # system variables
On TrueNAS SCALE
systemctl status chrony
journalctl -u chrony --no-pager -n 200
chronyc -N -n ntpdata 192.168.3.92
sudo tcpdump -ni any 'udp port 123 and host 192.168.3.92' -vv -c 6
Firewalls
Pi (UFW): sudo ufw allow proto udp from 192.168.3.0/24 to any port 123
OPNsense: allow LAN→this firewall UDP/123 if you serve time to LAN.
Ensure no NAT or WAN filtering is interfering with internal NTP.
Common pitfalls (and fixes)
OPNsense “restrict unusable” error
Use restrict A.B.C.D mask 255.255.255.0 ... (no CIDR).
“Address already in use” starting ntpd
Another ntpd instance is running or bound. Stop via GUI or kill before manual tests.
/dev/pps0 looks wrong (weird symlink) or ppstest fails
Make sure you used the right overlay (pps-gpio) for GPIO PPS, or that your USB GPS actually exposes a PPS device. Reboot after overlay changes.
NTP to the Internet is blocked upstream
Your Pi doesn’t need the public NTP if GPS/PPS is stable. If you want Internet sanity check with NTS, remember: NTS still ultimately uses UDP/123 for time packets (TLS is only for key exchange on TCP/4460). If UDP/123 is blocked outbound, public NTP (including NTS time) won’t work.
Wi-Fi/Ethernet IP confusion
Disable Wi-Fi (dtoverlay=disable-wifi) or set bindaddress in chrony to your Ethernet IP.
Complete example configs
/etc/chrony/chrony.conf (Pi)
# Stratum-1 GPS/PPS server
minsources 1
makestep 1.0 3
rtcsync
driftfile /var/lib/chrony/chrony.drift
logdir /var/log/chrony

# GPS via gpsd (NMEA on SHM 0) - adjust offset after calibration
refclock SHM 0 offset 0.5 delay 0.2 refid NMEA

# PPS device
refclock PPS /dev/pps0 refid PPS lock NMEA prefer

# Optional NTS sanity check
# server time.cloudflare.com iburst nts
# server nts.netnod.se      iburst nts

# Serve time to LAN only
allow 192.168.3.0/24

# Bind to Ethernet IP
bindaddress 192.168.3.92

# Local admin
bindcmdaddress 127.0.0.1
cmdport 323
cmdallow 127.0.0.1
OPNsense ntpd.conf excerpt (auto-generated; for reference)
tinker panic 0
tos orphan 15
tos maxclock 10

# Upstream (Pi)
server 192.168.3.92 iburst maxpoll 9

# Logging/paths
driftfile /var/db/ntpd.drift
statsdir /var/log/ntp
logconfig =syncall +clockall +peerall +sysall

# Access restrictions (note the mask syntax)
restrict source  kod limited nomodify noquery notrap
restrict default kod limited nomodify noquery notrap nopeer
restrict -6 default kod limited nomodify noquery notrap nopeer
restrict 127.0.0.1 kod limited nomodify notrap nopeer
restrict ::1      kod limited nomodify notrap nopeer
restrict 192.168.3.0 mask 255.255.255.0 nomodify notrap nopeer
FAQ
Q: Do I need public NTP/NTS once GPS/PPS is up?
No. GPS/PPS makes your Pi an authoritative stratum-1. Public servers are helpful for initial sanity, but not required.
Q: How long until PPS goes steady?
Expect a few minutes for satellite lock; jitter tightens over the first hours. Re-calibrate the NMEA offset after a day or two.
Q: Can OPNsense also use a local PPS?
Yes, but it’s redundant if the Pi already provides PPS-disciplined time. Keeping one stratum-1 simplifies life.
License
MIT. PRs welcome—especially for other GPS models and wiring photos.
Quick copy/paste (bring everything up)
# On the Pi:
sudo apt update && sudo apt install -y chrony gpsd gpsd-clients pps-tools
echo 'dtoverlay=pps-gpio,gpiopin=18' | sudo tee -a /boot/firmware/config.txt
sudo systemctl enable --now gpsd
sudo tee /etc/chrony/chrony.conf >/dev/null <<'EOF'
minsources 1
makestep 1.0 3
rtcsync
driftfile /var/lib/chrony/chrony.drift
logdir /var/log/chrony
refclock SHM 0 offset 0.5 delay 0.2 refid NMEA
refclock PPS /dev/pps0 refid PPS lock NMEA prefer
allow 192.168.3.0/24
bindaddress 192.168.3.92
bindcmdaddress 127.0.0.1
cmdport 323
cmdallow 127.0.0.1
EOF
sudo systemctl enable --now chrony
sudo ufw allow proto udp from 192.168.3.0/24 to any port 123 || true
chronyc sources -v && chronyc tracking
I
