# Minimal Pi-hole On Raspberry Pi Zero W

Headless Raspberry Pi OS / Raspbian 13 (Trixie), 32-bit, for the original
Raspberry Pi Zero W: **Pi-hole + Unbound + SSH + onboard Wi-Fi**.
No desktop, printing, Bluetooth, camera, or development toolchain.
Reduced from **1,665 to 501 packages**.

## Files

- `packages.tsv`: final installed packages, versions, and architectures.
- `manual.txt`: packages explicitly retained rather than automatically installed.
- `initial.txt`, `final.txt`, `removed.txt`: sorted `package:architecture` lists
  before cleanup, afterward, and removed (1,665 / 501 / 1,164 entries).
- `system.txt`: OS, kernel, Pi-hole versions, repositories, and package holds.

## Clean A Fresh Installation

1. Set up the matching 32-bit OS, SSH, Wi-Fi, Pi-hole, and Unbound. Verify access and DNS, then protect their dependencies:

   ```bash
   sudo apt-mark manual pihole-meta unbound unbound-anchor ssh \
     network-manager wpasupplicant firmware-brcm80211 \
     linux-image-rpi-v6 raspi-firmware initramfs-tools systemd-timesyncd
   ```

  Back up Pi-hole (Teleporter), Unbound, networking, boot, and log2ram configuration
  separately. Log2ram is active on this setup but absent from the APT manifest.

2. In Bash, from this repository's directory on the Pi, compare **package names
   only**:

   ```bash
   dpkg-query -W -f='${db:Status-Status}\t${Package}\n' |
     awk -F '\t' '$1 == "installed" {print $2}' |
     LC_ALL=C sort -u > fresh.txt

   diff -u <(cut -d: -f1 initial.txt | LC_ALL=C sort -u) fresh.txt

   LC_ALL=C comm -23 fresh.txt \
     <(cut -d: -f1 initial.txt | LC_ALL=C sort -u) > review.txt

   LC_ALL=C comm -12 fresh.txt \
     <(cut -d: -f1 removed.txt | LC_ALL=C sort -u) > delete.txt
   ```

   `fresh.txt` records everything currently installed. The diff against
   `initial.txt` shows added (`+`) and missing (`-`) names. **`diff` exit status 1
   means differences found**; 0 means identical, 2 means an error.

   `review.txt` contains only names new to the original installation.
   `delete.txt` contains previously removed names still
   installed on this Pi; absent names are excluded from both lists.
   Kernel versions in names still differ; added/missing names do not prove renames.

3. Edit the smaller `review.txt` first: leave **only new packages you approve for
   removal**, one name per line. Delete entries you want to keep. Save and exit
  before merging it with the historical candidates.

   Then merge, deduplicate, and simulate. Historical removals are candidates,
   not proof that a package is unnecessary on a new release. Remove any known
   exceptions from `delete.txt` before running the simulation:

   ```bash
   LC_ALL=C sort -u -o delete.txt delete.txt review.txt
   nano delete.txt
   xargs -r -a delete.txt sudo apt-get -s purge --autoremove
   ```

   Inspect the complete simulated plan, including dependency removals. Only when
   it is acceptable, feed the same reviewed list to the real purge:

   ```bash
  xargs -r -a delete.txt sudo apt-get purge --autoremove
   ```

   Keep the running kernel, boot firmware, SSH, Wi-Fi, DNS, and swap dependencies.
   Pi-hole requires **binutils**, even without a compiler. Keep Netplan/cloud-init
   if they manage networking; verify a system-owned Wi-Fi password before removing
   GNOME keyring. Manual package marking does not prevent dependency conflicts.

4. Verify package health, SSH, Wi-Fi autoconnect, DNS, and the boot files after
   cleanup. Reboot, reconnect over SSH, and check DNS again. Save the new final
   package list for the next comparison.
