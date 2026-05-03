# VM Network Manual Setup

Initial VM network configuration is done manually before Ansible can reach the hosts.

## Network Overview

Each VM has two NICs:

| Connection Name | Type        | Subnet            | Purpose                                          |
| --------------- | ----------- | ----------------- | ------------------------------------------------ |
| `shared`        | Shared (NAT) | 192.168.64.0/24  | Host SSH access, internet                        |
| `internal`      | Host-Only   | 10.220.100.0/24   | Cluster traffic (Patroni / etcd / replication)   |

- `enp0s1` → `shared` (DHCP, 192.168.64.0/24)
- `enp0s2` → `internal` (static, 10.220.100.{11,12,13}/24)

Clients reach the writable node through HAProxy on `oss-cor` (10.220.100.13); no floating VIP is used.

## oss-db1 (10.220.100.11)

```bash
hostnamectl set-hostname oss-db1
nmcli con add type ethernet ifname enp0s1 con-name shared ipv4.method auto connection.autoconnect yes
nmcli con add type ethernet ifname enp0s2 con-name internal ipv4.method manual ipv4.addresses 10.220.100.11/24 connection.autoconnect yes
nmcli con up shared
nmcli con up internal
```

## oss-db2 (10.220.100.12)

```bash
hostnamectl set-hostname oss-db2
nmcli con add type ethernet ifname enp0s1 con-name shared ipv4.method auto connection.autoconnect yes
nmcli con add type ethernet ifname enp0s2 con-name internal ipv4.method manual ipv4.addresses 10.220.100.12/24 connection.autoconnect yes
nmcli con up shared
nmcli con up internal
```

## oss-cor (10.220.100.13)

```bash
hostnamectl set-hostname oss-cor
nmcli con add type ethernet ifname enp0s1 con-name shared ipv4.method auto connection.autoconnect yes
nmcli con add type ethernet ifname enp0s2 con-name internal ipv4.method manual ipv4.addresses 10.220.100.13/24 connection.autoconnect yes
nmcli con up shared
nmcli con up internal
```

## SSH Port Change

All VMs use port **11122** instead of the default 22.

```bash
setenforce 0
dnf install -y iptables-services

sed -i 's/^#Port 22/Port 11122/' /etc/ssh/sshd_config
systemctl restart sshd

firewall-cmd --permanent --add-port=11122/tcp
firewall-cmd --permanent --remove-service=ssh
firewall-cmd --reload

sed -i 's/--dport 22/--dport 11122/' /etc/sysconfig/iptables
systemctl restart iptables
```

> Verify the new port works (`ssh -p 11122 ...` from another terminal) **before** closing the original SSH session — `--remove-service=ssh` drops port 22, so a misconfigured sshd will lock you out.
