# 3x-ui MikroTik RouterOS Container

This repository contains prebuilt MikroTik RouterOS container `.tar` images for running **3x-ui** on MikroTik RouterOS.

3x-ui is a web-based Xray management panel. This package is intended for MikroTik devices or MikroTik CHR systems that support RouterOS Containers.

---

## Files

Example repository structure:

```text
.
├── README.md
├── 3xui-v*.7z
└── 3xui-v*-arm64.7z
```

Recommended image names (inside 7z files):

| File | Target device |
|---|---|
| `3xui-v*.tar` | MikroTik CHR / x86_64 |
| `3xui-v*-arm64.tar` | ARM64 MikroTik devices |

You can rename the `.tar` files, but make sure the file name used in RouterOS commands matches the uploaded file.

---

## Requirements

Before deploying, make sure your MikroTik device supports containers.

Minimum requirements:

- RouterOS v7.x
- MikroTik `container` package installed
- Container mode enabled in `/system/device-mode`
- Enough RAM and storage
- External USB/SSD storage is strongly recommended
- Internet access from the container network
- Correct image for your CPU architecture

---

## Important Security Notice

Do **not** expose the 3x-ui panel directly to the public internet without protection.

After first login:

1. Change the default username and password immediately.
2. Use a strong password.
3. Restrict access to the panel port using MikroTik firewall rules.
4. Prefer access through VPN, WireGuard, ZeroTier, or a trusted admin IP.
5. Keep RouterOS updated.
6. Keep the container image updated.
7. Do not run unknown or untrusted container images.

Default 3x-ui credentials may be:

```text
Username: admin
Password: admin
```

or:

```text
Username: admin
Password: <empty>
```

Try `admin/admin` first. If it does not work, try username `admin` with an empty password.

---

## Step 1 — Check RouterOS Version

Run:

```routeros
/system/resource/print
/system/package/print
```

You should be using RouterOS v7.

---

## Step 2 — Install the Container Package

Check installed packages:

```routeros
/system/package/print
```

You should see the `container` package.

If the package is not installed:

1. Go to the MikroTik download page.
2. Download the **Extra Packages** archive for your RouterOS version and CPU architecture.
3. Extract the archive on your computer.
4. Upload the `container-*.npk` file to the router.
5. Reboot the router.

After reboot, verify again:

```routeros
/system/package/print
```

---

## Step 3 — Enable Container Mode

Run:

```routeros
/system/device-mode/update container=yes
```

RouterOS will require physical confirmation.

Depending on your device, confirm by:

- pressing the physical reset/mode button within the countdown, or
- performing a cold reboot / power cycle

After the router reboots, verify:

```routeros
/system/device-mode/print
```

You should see:

```text
container: yes
```

---

## Step 4 — Prepare Storage

External USB/SSD storage is recommended.

Check disks:

```routeros
/disk/print
```

If your disk appears as `usb1`, create directories:

```routeros
/file/make-directory usb1/containers
/file/make-directory usb1/containers/3xui
/file/make-directory usb1/containers/3xui/root
```

If you do not have external storage, you can use internal storage:

```routeros
/file/make-directory containers
/file/make-directory containers/3xui
/file/make-directory containers/3xui/root
```

For the rest of this README, the examples use external storage:

```text
usb1/containers
```

If you use internal storage, replace:

```text
usb1/containers
```

with:

```text
containers
```

---

## Step 5 — Create Container Network

Create a dedicated bridge:

```routeros
/interface/bridge/add name=containers
```

Create a virtual Ethernet interface for the 3x-ui container:

```routeros
/interface/veth/add name=veth-3xui address=172.19.0.2/24 gateway=172.19.0.1
```

Add the VETH interface to the container bridge:

```routeros
/interface/bridge/port/add bridge=containers interface=veth-3xui
```

Assign an IP address to the container bridge:

```routeros
/ip/address/add address=172.19.0.1/24 interface=containers
```

Add NAT so the container can access the internet:

```routeros
/ip/firewall/nat/add chain=srcnat src-address=172.19.0.0/24 action=masquerade comment="NAT for containers"
```

Allow forwarding if your firewall blocks container traffic:

```routeros
/ip/firewall/filter/add chain=forward src-address=172.19.0.0/24 action=accept comment="Allow containers outbound"
```

---

## Step 6 — Upload the `.tar` Image

Upload the correct `.tar` file to your MikroTik using WinBox, WebFig, FTP, or SCP.

For MikroTik CHR / x86_64:

```text
usb1/containers/3xui-v*.tar
```

For ARM64 MikroTik devices:

```text
usb1/containers/3xui-v*-arm64.tar
```

---

## Step 7A — Deploy AMD64 / CHR Version

Use this version for MikroTik CHR or x86_64 RouterOS.

```routeros
/container/add \
    file=usb1/containers/3x-ui-routeros-amd64.tar \
    interface=veth-3xui \
    root-dir=usb1/containers/3xui/root \
    start-on-boot=yes \
    logging=yes
```

---

## Step 7B — Deploy ARM64 Version

Use this version for ARM64 MikroTik devices.

```routeros
/container/add \
    file=usb1/containers/3x-ui-routeros-arm64.tar \
    interface=veth-3xui \
    root-dir=usb1/containers/3xui/root \
    start-on-boot=yes \
    logging=yes
```

---

## Step 8 — Start the Container

List containers:

```routeros
/container/print
```

Start the container:

```routeros
/container/start 0
```

Check details:

```routeros
/container/print detail
```

Check logs:

```routeros
/log/print where topics~"container"
```

---

## Step 9 — Access 3x-ui Panel

The container IP is:

```text
172.19.0.2
```

The default 3x-ui panel port is commonly:

```text
2053
```

Try opening:

```text
http://172.19.0.2:2053
```

If you want to access the panel from your LAN through the router IP, add a destination NAT rule.

Example: expose the panel on RouterOS port `2053`:

```routeros
/ip/firewall/nat/add \
    chain=dstnat \
    protocol=tcp \
    dst-port=2053 \
    action=dst-nat \
    to-addresses=172.19.0.2 \
    to-ports=2053 \
    comment="Forward 3x-ui panel to container"
```

Then open:

```text
http://ROUTER_IP:2053
```

Example:

```text
http://192.168.88.1:2053
```

---

## Default Login

Try:

```text
Username: admin
Password: admin
```

If that does not work, try:

```text
Username: admin
Password:
```

The second option means username `admin` with an empty password.

Change the login credentials immediately after first login.

---

## Recommended Firewall Protection

### Protect Panel Access from WAN

Do not allow public access to the 3x-ui panel.

Allow only your trusted admin IP:

```routeros
/ip/firewall/filter/add \
    chain=input \
    protocol=tcp \
    dst-port=2053 \
    src-address=YOUR_ADMIN_IP \
    action=accept \
    comment="Allow 3x-ui panel from admin IP"

/ip/firewall/filter/add \
    chain=input \
    protocol=tcp \
    dst-port=2053 \
    action=drop \
    comment="Drop public access to 3x-ui panel"
```

Replace:

```text
YOUR_ADMIN_IP
```

with your real trusted IP address.

---

### Protect Forwarded Panel Access

If you use `dstnat` to forward the panel to the container, protect the `forward` chain too:

```routeros
/ip/firewall/filter/add \
    chain=forward \
    protocol=tcp \
    dst-address=172.19.0.2 \
    dst-port=2053 \
    src-address=YOUR_ADMIN_IP \
    action=accept \
    comment="Allow forwarded 3x-ui panel from admin IP"

/ip/firewall/filter/add \
    chain=forward \
    protocol=tcp \
    dst-address=172.19.0.2 \
    dst-port=2053 \
    action=drop \
    comment="Drop other forwarded access to 3x-ui panel"
```

---

## Optional: Expose 3x-ui Service Ports

If your 3x-ui/Xray inbound uses ports such as `443`, `8443`, or another custom port, forward only the ports you actually use.

Example forwarding TCP `443`:

```routeros
/ip/firewall/nat/add \
    chain=dstnat \
    protocol=tcp \
    dst-port=443 \
    action=dst-nat \
    to-addresses=172.19.0.2 \
    to-ports=443 \
    comment="Forward 3x-ui service TCP 443"
```

Example forwarding UDP `443`:

```routeros
/ip/firewall/nat/add \
    chain=dstnat \
    protocol=udp \
    dst-port=443 \
    action=dst-nat \
    to-addresses=172.19.0.2 \
    to-ports=443 \
    comment="Forward 3x-ui service UDP 443"
```

For another inbound port, replace `443` with your real port.

---

## Stop, Start, Restart

Start:

```routeros
/container/start 0
```

Stop:

```routeros
/container/stop 0
```

Restart:

```routeros
/container/stop 0
/container/start 0
```

---

## Remove Container

Stop the container first:

```routeros
/container/stop 0
```

Remove it:

```routeros
/container/remove 0
```

The root directory may remain on disk. Remove it manually only if you no longer need the data.

---

## Update Container Image

To update the container image:

1. Stop the old container.
2. Backup your 3x-ui data if needed.
3. Remove the old container entry.
4. Upload the new `.tar` image.
5. Add the new container again.
6. Use the same `root-dir` if you want to preserve existing data.

Example:

```routeros
/container/stop 0
/container/remove 0
```

Then deploy the new `.tar` again using the same deployment command.

---

## Backup Data

If your container stores 3x-ui data inside:

```text
usb1/containers/3xui/root
```

backup this directory before removing, updating, or replacing the container.

You can copy it using WinBox, FTP, SCP, or MikroTik file tools.

---

## Troubleshooting

### Container Does Not Start

Check:

```routeros
/container/print detail
/log/print where topics~"container"
```

Common causes:

- Wrong CPU architecture image
- Missing `container` package
- Container mode not enabled
- Not enough RAM
- Not enough disk space
- Invalid `root-dir`
- Corrupted `.tar` upload
- Unsupported RouterOS/device architecture

---

### No Internet Inside Container

Check NAT:

```routeros
/ip/firewall/nat/print
```

Make sure this rule exists:

```routeros
/ip/firewall/nat/add chain=srcnat src-address=172.19.0.0/24 action=masquerade
```

Check VETH configuration:

```routeros
/interface/veth/print detail
```

Expected configuration:

```text
name=veth-3xui
address=172.19.0.2/24
gateway=172.19.0.1
```

Check bridge IP:

```routeros
/ip/address/print
```

Expected bridge IP:

```text
172.19.0.1/24
```

---

### Cannot Open 3x-ui Panel

Check that the container is running:

```routeros
/container/print
```

Check logs:

```routeros
/log/print where topics~"container"
```

Check that the panel port is correct. The common default is:

```text
2053
```

Try:

```text
http://172.19.0.2:2053
```

If accessing from LAN through the router IP, make sure you added the `dstnat` rule.

---

### Login Does Not Work

Try:

```text
admin / admin
```

or:

```text
admin / empty password
```

If both fail, check your image build documentation or reset the 3x-ui panel credentials from inside the container if shell access is available.

---


## Notes

- RouterOS Containers are not managed with Docker commands.
- MikroTik uses its own `/container` command system.
- Use external storage when possible.
- Do not expose the 3x-ui admin panel publicly.
- Always change the default login.
- Always keep a backup before updating or replacing the container.
- Only expose service ports that you actually need.
- Keep RouterOS and the container image updated.

---

## Disclaimer

This project is provided as-is.

You are responsible for securing your MikroTik router, RouterOS firewall, container network, exposed ports, and 3x-ui panel credentials.

Use this container only on devices and networks where you have permission to deploy and operate it.
