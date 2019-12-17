# Installing coreos with PXE

## Setup the nodes

Ensure that BIOS settings enable PXE boot.
- some laptops enable PXE automatically for wake-on-lan.
- To send wake-on-lan packets: `sudo dnf install wol`

Set up ./etc/hosts and ./etc/dhcp/dhcpd.conf with node addresses.

## Setup console

Use console (control laptop, Fedora 31) as temporary DHCP, TFTP and HTTP server.

- Disable LAN router DHCP

```
cp ./etc/hosts /etc/hosts
cp ./etc/dhcp/dhcpd.conf /etc/dhcp
systemctl start dhcpd tftp httpd
systemctl stop firewalld
```

- Follow: https://coreos.com/os/docs/latest/booting-with-pxe.html
  - See [dhcpd.conf](./etc/dhcp/dhcpd.conf)
  - Don't forget to set a public key for SSH
  - Need fcct for config: https://github.com/coreos/fcct/blob/master/docs/getting-started.md
  - GPG keys: https://coreos.com/os/docs/latest/verify-images.html
- Copy pxe-config.ign to /var/www/html (had problems with tftp URLs on some nodes)

- Reboot nodes to PXE boot

## Install to disk (wipes disk!)

See https://coreos.com/os/docs/latest/installing-to-disk.html

```
export HOSTS=<ip addresses for nodes>
```

Remove /dev/mapper entries for old Fedora LVM mounts:

```
for h in $HOSTS; do ssh core@$h sudo lvremove -f '/dev/mapper/*'; done
```

Replace 192.168.1.100 with console address here:

```
for h in $HOSTS; do ssh core@$h wget http://192.168.1.100/pxe-config.ign; done
for h in $HOSTS; do ssh core@$h sudo coreos-install -d /dev/sda -i pxe-config.ign&
wait
for h in $HOSTS; do 
    ssh core@$h<<EOF 
touch .hushlogin
sudo su
hostnamectl set-hostname $h
systemctl mask --now sleep.target suspend.target hibernate.target hybrid-sleep.target
echo -e '[Login]\nHandleLidSwitchExternalPower=ignore' > /etc/systemd/logind.conf
EOF
for h in $HOSTS; do cat ./etc/hosts | ssh -T $h 'sudo tee /etc/hosts > /dev/null'; done
done
```

## Shutdown temp DHCP server
```
systemctl stop dhcpd tftpd httpd
systemctl start firewalld
```

- Configure router to assign same IP addresses as dhcpd.conf
- Re-enable router dhcp.
- Reboot everything!
