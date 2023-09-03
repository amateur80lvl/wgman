# Wireguard configuration management suite

These scripts generate wireguard configuration files and HTML files with QR codes for mobile clients.

## How to use

* Fetch this suite
* Install python packages `pyyaml` and `qrcode`
* Сhange directory to `wgman`
* Make subdirectory with the name of your server and copy `config.yaml.sample` into it:
```bash
$ mkdir sample-server
$ cp config.yaml.sample sample-server/config.yaml
```
* Edit configuration file. It's pretty simple:
```yaml
subnet: 10.10.0.0
subnet_bits: 24
server_private_address: 10.10.0.1
client_address_start: 2

server_public_address: 1.2.3.4
server_port: 443

use_preshared_key: true

default_route: true
dns: 1.1.1.1
```

Minimal required changes are `server_public_address` and `server_port`. All the rest, such as `default_route`, `dns`, `use_preshared_key` is global, for all clients. This is a flaw but you can edit `config.yaml` before creating new client,
or edit generated configuration. Although, in the latter case it's not possible to update QR code easily.

So, we have server directory with `config.yaml`. It's time to create server configuration for wireguard:
```bash
$ ./create-server sample-server
```

Here's what we get:
```bash
$ ls sample-server/
```
```
config.yaml  private-key  public-key  wg0.conf
```

`wg0.conf` is the configuration for server's interface. Copy or symlink it to /etc/wireguard and then, if you're an involuntarily systemd fan as me:
```bash
systemctl enable wg-quick@wg0
```

It's not necessary to start wg0 right now because configuration is half-way:
```bash
$ less sample-server/wg0.conf
```
```
[Interface]
PrivateKey = AAbwXXEFu/Hy1zncqri+dsTmZEdEpr5SwWlF0bdsdks=  # server private key
ListenPort = 443
Address = 10.10.0.1
```

We have to create clients. A couple, for instance:
```bash
$ ./create-client sample-server client-one
$ ./create-client sample-server client-two
```

Now we have:
```bash
$ ls sample-server/
```
```
client-one.conf           client-two.conf           config.yaml
client-one.html           client-two.html           ipaddr-map
client-one.preshared-key  client-two.preshared-key  private-key
client-one.private-key    client-two.private-key    public-key
client-one.public-key     client-two.public-key     wg0.conf
```

Server configuration file now looks like this:
```bash
$ less sample-server/wg0.conf
```
```
[Interface]
PrivateKey = AAbwXXEFu/Hy1zncqri+dsTmZEdEpr5SwWlF0bdsdks=  # server private key
ListenPort = 443
Address = 10.10.0.1

[Peer]
PublicKey = KcNIa1/Tbv43nWZ+GEXfmr+cNL951yoduX7ucwtB4FM=  # client-two public key
AllowedIPs = 10.10.0.3/32  # client-two IP address
PresharedKey = f734D81tizY35ypm1urnUFlKhxMAKp1cCpanWfuuhSA=

[Peer]
PublicKey = yRrVT/Hgo4uOFejvSATHKRzAcAmpWKO0zw25j/lLDBA=  # client-one public key
AllowedIPs = 10.10.0.2/32  # client-one IP address
PresharedKey = pzEP1x4b3g50AqzFiI9nsLAu+zUjjg+KcqjJuOO/jLU=
```

Here's one of clients configuration:
```bash
$ less sample-server/client-one.conf
```
```
[Interface]
PrivateKey = oJs9Df5oWhqMiKOv/77SUTh6n16F5i2BqJW4bQ5/ZE8=  # client private key
Address = 10.10.0.2/32
DNS = 1.1.1.1

[Peer]
Endpoint = 1.2.3.4:443
PublicKey = C/hqK9Bza7m13KSgpqykb2/IsXaw+W2I0ii5/9xOZXE=  # server public key
PersistentKeepalive = 15  # we need this if we're behind a firewall
AllowedIPs = 0.0.0.0/0  # default route
PresharedKey = pzEP1x4b3g50AqzFiI9nsLAu+zUjjg+KcqjJuOO/jLU=
```

That's all. Don't forget to restart server interface after adding new clients. Although you can do without restart
```bash
$ wg syncconf wg0 <(wg-quick strip wg0)
```
but you'll have to add routes manually then.
