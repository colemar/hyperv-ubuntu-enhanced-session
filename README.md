# Ubuntu 24.04 VM on Hyper-V with Enhanced Session Configuration

**Original Source:** The commands and procedure described in this file are adapted from the guide by **klo2k** on dev.to. Please refer to the original article for the full context and any updates:
[Create Ubuntu 24.04.2 VM in Hyper-V, with "Enhanced Session" RDP support (Windows 11, xrdp, development)](https://dev.to/klo2k/create-ubuntu-24042-vm-in-hyper-v-with-enhanced-session-rdp-support-windows-11-xrdp-1omk)

---

If you create the Virtual Machine using Hyper-V's Quick Create feature, you don't need the steps in this tutorial.

## Create a new Hyper-V Virtual Machine with Ubuntu 24.04 guest OS (or import/convert it from existing VM)

- Enable `Allow enhanced session mode` in Hyper-V Settings
- VM must be Generation 2
- Must enable Secure Boot with `Microsoft UEFI Certificate Authority`
- RAM: 6144 MB (if less then reduce zram-size below)
- Execute in an elevated Powershell:
  - `Set-VM -VMName 'vmname' -EnhancedSessionTransportType HvSocket`
  - `(Get-VM -VMName 'vmname').EnhancedSessionTransportType`
  - `Set-VMProcessor -VMName 'vmname' -ExposeVirtualizationExtensions $true`
  - `(Get-VMProcessor -VMName 'vmname').ExposeVirtualizationExtensions`
- Make sure `Require my password to log in` is enabled while installing guest OS.

## Run as root in guest OS terminal:

```bash
# Update system and install Hyper-V integration services
apt update && apt upgrade -y
apt install -y linux-tools-virtual linux-cloud-tools-virtual

# Fix KVP daemon log errors
mkdir -p /usr/libexec/hypervkvpd/
ln -sf /usr/sbin/hv_get_dhcp_info /usr/libexec/hypervkvpd/hv_get_dhcp_info
ln -sf /usr/sbin/hv_get_dns_info /usr/libexec/hypervkvpd/hv_get_dns_info

# Install and configure xrdp
apt install -y xrdp
cp -p /etc/xrdp/sesman.ini /etc/xrdp/sesman.ini.original
cp -p /etc/xrdp/xrdp.ini /etc/xrdp/xrdp.ini.original

# Enable vsock (hybrid configuration)
sed -i -e 's/^port=3389$/port=3389 vsock:\/\/-1:3389/g' /etc/xrdp/xrdp.ini

# Rename shared drives mount point
sed -i -e 's/FuseMountName=thinclient_drives/FuseMountName=shared-drives/g' /etc/xrdp/sesman.ini

# Create script for Ubuntu session environment
cat > /etc/xrdp/startubuntu.sh << EOF
#!/bin/sh
export GNOME_SHELL_SESSION_MODE=ubuntu
export XDG_CURRENT_DESKTOP=ubuntu:GNOME
exec /etc/xrdp/startwm.sh
EOF
chmod a+x /etc/xrdp/startubuntu.sh

# Use the startubuntu.sh script in sesman
sed -i -e 's/startwm/startubuntu/g' /etc/xrdp/sesman.ini

# Fix login black screen delay by blacklisting VMware module
echo "blacklist vmw_vsock_vmci_transport" > /etc/modprobe.d/blacklist-vmw_vsock_vmci_transport.conf

# Configure PAM for keyring unlocking
cat > /etc/pam.d/xrdp-sesman <<'EOT'
#%PAM-1.0
auth     required  pam_env.so readenv=1
auth     required  pam_env.so readenv=1 envfile=/etc/default/locale
@include common-auth
-auth    optional  pam_gnome_keyring.so
-auth    optional  pam_kwallet5.so

@include common-account

@include common-password

# Ensure resource limits are applied
session    required     pam_limits.so
# Set the loginuid process attribute.
session    required     pam_loginuid.so
# Update wtmp/lastlog
session    optional     pam_lastlog.so quiet
@include common-session
-session optional  pam_gnome_keyring.so auto_start
-session optional  pam_kwallet5.so auto_start
EOT

# Apply changes and restart service (Do this *before* rebooting)
systemctl restart xrdp

# Optional: Show boot log instead of splash screen and reduce grub fail timeout
cp -p /etc/default/grub /etc/default/grub.default
sed -i -e 's/GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT=""/g' /etc/default/grub
echo 'GRUB_RECORDFAIL_TIMEOUT=3' >> /etc/default/grub
update-grub

# Optional: Replace swap file with ZRAM (adjust zram-size as needed, e.g., half of VM RAM)
apt install -y systemd-zram-generator
cp -p /etc/systemd/zram-generator.conf /etc/systemd/zram-generator.conf.original
cat > /etc/systemd/zram-generator.conf <<'EOT'
[zram0]
zram-size = 4096 # Example: 4GB for a 6-8GB VM
compression-algorithm = zstd
EOT
systemctl daemon-reload
systemctl restart systemd-zram-setup@zram0
# Disable and remove old swap file
cp -p /etc/fstab /etc/fstab.original
sed -i -e 's@^/swap.img@#/swap.img@g' /etc/fstab
swapoff /swap.img
rm /swap.img

# Optional: Switch to the VNC Backend.
# In Feb 2025 one of the Ubuntu update caused a performance regression with the Xorg backend (even on 1920x1080)
# Install TigerVNC
apt install -y tigervnc-standalone-server tigervnc-xorg-extension
# Configure tigervnc backend
cp -p /etc/xrdp/sesman.ini /etc/xrdp/sesman.ini.Xorg
sed -z -i -e \
's/'\
'\(\[Xvnc\].*\nparam=96\n\)\n\[/'\
'\1param=-CompareFB\nparam=1\nparam=-ZlibLevel\nparam=0\nparam=-geometry\nparam=1920x1080\n\n[/'\
'gi' /etc/xrdp/sesman.ini
# Remove Xorg backend
cp -p /etc/xrdp/xrdp.ini /etc/xrdp/xrdp.ini.Xorg
sed -z -i -e \
's/'\
'\[Xorg\].*\ncode=20\n\n\[/'\
'[/'\
'gi' /etc/xrdp/xrdp.ini

reboot
```
