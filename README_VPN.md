
VPN server install from:
https://kifarunix.com/install-and-setup-openvpn-server-on-fedora-29-centos-7/

```
sudo dnf install -y openvpn easy-rsa
mkdir /etc/openvpn/easy-rsa; cd /etc/openvpn/easy-rsa
cp -air /usr/share/easy-rsa/3/* .
./easyrsa init-pki && ./easyrsa build-ca
# passphrase: ca0. + 1 ; CN Alan Conway. Cert /etc/openvpn/easy-rsa/pki/ca.crt
./easyrsa gen-dh # DH key (long)  /etc/openvpn/easy-rsa/pki/dh.pem
./easyrsa build-server-full server nopass # same pass
./easyrsa build-client-full client nopass # same pass
./easyrsa gen-crl # revocation cert  /etc/openvpn/easy-rsa/pki/crl.pem
openvpn --genkey --secret pki/ta.key
cp -rp /pki/{ca.crt,dh.pem,ta.key,issued,private} /etc/openvpn/server/
cp /usr/share/doc/openvpn/sample/sample-config-files/server.conf /etc/openvpn/server/
# Edits to server.conf as per web page
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl --system
firewall-cmd --add-port=1194/udp --permanent
firewall-cmd --add-masquerade --permanent
# Find enpxxx device with: ip route get 8.8.8.8
firewall-cmd --permanent --direct --passthrough ipv4 -t nat -A POSTROUTING -s 172.16.0.0/24 -o enp0s25 -j MASQUERADE
firewall-cmd --reload
nnsystemctl enable --now openvpn-server@server

# Copy to client, on client host
scp root@oscar3:/etc/openvpn/easy-rsa/pki/{ca.crt,issued/client.crt,private/client.key,ta.key} ~/.cert
restorecon -R -v ~/.cert
dnf install -y openvpn
# client config is in ~/etc/home-occ.ovpn
sudo nmcli connection import type openvpn file ~/etc/home-occ.ovpn

# Add static routes for access to hosts other than oscar3:
ssh oscar3 'for h in oscar1 oscar2; do  ssh -lroot $h ip route add 10.8.0.0/16 via 192.168.2.103; done'
```
