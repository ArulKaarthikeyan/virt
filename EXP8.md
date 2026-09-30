# Exp 8: Network Virtualization — OpenVSwitch and Linux Bridge

(Matches reference doc — Reg.No: 2303917720521020)

---

## PART 1: Linux Bridge

**1. Check current config**

```bash
ifconfig
```

**2. Install bridge utilities + load kernel modules**

```bash
sudo apt install bridge-utils -y
sudo modprobe bridge
sudo modprobe br_netfilter
```

**3. Edit netplan config**

```bash
sudo nano /etc/netplan/01-network-manager-all.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
  bridges:
    br-cloud:
      interfaces: [enp0s3]
      addresses: [10.0.2.15/24]
      routes:
        - to: default
          via: 10.0.2.2
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

> Ubuntu 20.04+: use `routes:` (above) — `gateway4:` is deprecated and prints a warning.
> Ubuntu 18.04 / matches reference doc: `gateway4: 10.0.2.2` also works, no warning.

**4. Apply and restart**

```bash
sudo systemctl enable systemd-networkd
sudo netplan apply
sudo systemctl restart NetworkManager
```

**5. Verify interfaces**

```bash
ip a
```

**6. Verify routing table**

```bash
route -n
brctl show
```

Expected: `br-cloud` holds the IP, `enp0s3` is its member (no IP of its own).

**7. Alternative — manual bridge (no netplan)**

```bash
sudo brctl addbr br-cloud-new
sudo brctl addif br-cloud-new enp0s3
sudo ifconfig enp0s3 0
sudo ifconfig br-cloud-new 10.0.2.15 netmask 255.255.255.0
sudo route add default gw 10.0.2.2 dev br-cloud-new
ip a
route -n
brctl show
```

---

## PART 2: OpenVSwitch (OVS)

**1. Become root**

```bash
sudo -i
```

**2. Create 2 namespaces**

```bash
ip netns add VRF1
ip netns add VRF2
ip netns list
```

**3. Create 4 veth pairs**

```bash
ip link add veth1 type veth peer name veth1-br
ip link add veth2 type veth peer name veth2-br
ip link add veth3 type veth peer name veth3-br
ip link add veth4 type veth peer name veth4-br
```

**4. Check links**

```bash
ip link
```

**5. Assign veth ports to namespaces**

```bash
ip link set veth1 netns VRF1
ip link set veth2 netns VRF1
ip link set veth3 netns VRF2
ip link set veth4 netns VRF2
```

Check:

```bash
ip netns exec VRF1 ip link
```

**6. Assign IP addresses**

```bash
ip netns exec VRF1 ifconfig veth1 10.10.10.1/24 up
ip netns exec VRF2 ifconfig veth3 10.10.10.2/24 up
```

Check:

```bash
ip netns exec VRF1 ifconfig
ip netns exec VRF2 ifconfig
```

**7. Check default routing table per namespace**

```bash
ip netns exec VRF1 route -n
ip netns exec VRF2 route -n
```

**8. Install OpenVSwitch**

```bash
apt install openvswitch-switch openvswitch-common -y
```

**9. Check/start service**

```bash
sudo systemctl status openvswitch-switch
sudo systemctl start openvswitch-switch
```

**10. Add bridge and ports**

```bash
ovs-vsctl add-br vSwitch1
ovs-vsctl add-port vSwitch1 veth2-br
ovs-vsctl add-port vSwitch1 veth4-br
ovs-vsctl show
```

**11. Bring ports up + assign switch IP**

```bash
ifconfig veth2-br up
ifconfig veth4-br up
ifconfig vSwitch1 10.10.10.5/24 up
```

**12. Ping test across namespaces via switch**

```bash
ip netns exec VRF1 ping 10.10.10.2
```

Expect 0% packet loss.

**13. (Optional) Enable Spanning Tree Protocol**

```bash
ovs-vsctl set bridge vSwitch1 stp_enable=true
ovsdb-client dump
```

**14. Verify MAC address match**

```bash
ovs-appctl fdb/show vSwitch1
ip netns exec VRF2 ifconfig
ip netns exec VRF1 ifconfig
```

Compare MACs in the OVS `fdb` table to `ether` addresses shown in each namespace's `ifconfig` — they must match.

**Result:** Network virtualization performed successfully using OpenVSwitch and Linux Bridge.

---

### Note

The reference only attaches `veth2-br`/`veth4-br` to the switch and only assigns IPs to `veth1`/`veth3`. If your ping in step 12 doesn't go through, also add and bring up `veth1-br`/`veth3-br`:

```bash
ovs-vsctl add-port vSwitch1 veth1-br
ovs-vsctl add-port vSwitch1 veth3-br
ifconfig veth1-br up
ifconfig veth3-br up
```
