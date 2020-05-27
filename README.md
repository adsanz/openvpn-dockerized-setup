# Docker-compose VPN - debian based systems

This repo contains the docker-compose configuration for a secured, speed-optimized openvpn ce setup + openvpn monitor (Tested on Ubuntu 20.04/18.04)

## Client requirements

* If linux openvpn > 2.4.0
* If mac tunnelblink >= 3.8
* No windows has been tested.

## Server requirements
* Docker CE >= 19.03
* Docker-compose >= 1.25.0
* If Prod
  * UFW
  * nginx-extras
  * apache2-utils

## First run 

### Configure IPtables (NOT FOR PROD)
Because openvpn needs IPtables to work... The next line needs to be added on `/etc/default/docker` 

```
DOCKER_OPTS="--iptables=true"
```

Then restart docker service.

If no /etc/defaults/docker is found, you should edit the systemd file usually at: `/lib/systemd/system/docker.service` and add `--iptables=true` at the end of the ExecStart command. It should look something like this:
```
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --iptables=true
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
```
After that run:
```
systemctl daemon-reload && systemctl restart docker
```

### Configure Prod ready
Docker gives 0 **** about UFW and usually ignores all those rules, so in this case if you wan't to use UFW (which you should) edit either `/etc/default/docker` or `/lib/systemd/system/docker.service` and add `--iptables=false` issue the next comands:

```
systemctl daemon-reload && systemctl restart docker
```
Then run this:
```
ufw deny incoming
ufw allow outgoing
sed -i -e 's/DEFAULT_FORWARD_POLICY="DROP"/DEFAULT_FORWARD_POLICY="ACCEPT"/g' /etc/default/ufw
iptables -t nat -A POSTROUTING ! -o docker0 -s 172.17.0.0/16 -j MASQUERADE
ufw allow 22/tcp 
ufw allow 1194/udp
ufw allow 80/tcp
ufw enable
```

Install nginx extras:
```
apt install nginx-extras
```

Configure Basic-auth:
```
apt install apache2-utils
touch /etc/nginx/.htpasswd
htpasswd /etc/nginx/.httpaswd MYUSER
```

And configure a site like this:

```
server {
    server_name _;
    listen 80;
    location / {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://127.0.0.1:8080;
    }
}
```

With this set-up you'll have secured the management interface.

### Docker compose starting configuration

To init the basic configuration:
```
# Init set-up with CBC usage - more compatibility with older openvpn clients
docker-compose run --rm openvpn ovpn_genconfig \
  -u udp://MYVPN.PUBLIC.IP.OR.DOMAIN \
  -n 1.1.1.1 \
  -a SHA512 \
  -s 10.0.0.0/24 \
  -r 10.0.0.0/24 \
  -C AES-256-CBC \
  -N \
  -m 24000 \
  -n 8.8.8.8 \
  -b \
  -T TLS-DHE-RSA-WITH-AES-256-CBC-SHA256 \
  -f 0 \
  -e "mssfix 0" \
  -e "txqueuelen 1000" \
  -e "management 0.0.0.0 5555" \
  -e "tls-version-min 1.2" 

# Init set-up with GCM usage > Openvpn 2.4 - this setup is way faster due to GCM
docker-compose run --rm openvpn ovpn_genconfig \
  -u udp://MYVPN.PUBLIC.IP.OR.DOMAIN \
  -a SHA512 \
  -n 1.1.1.1 \
  -s 10.0.0.0/24 \
  -r 10.0.0.0/24 \
  -C AES-256-GCM \
  -N \
  -m 24000 \
  -n 8.8.8.8 \
  -b \
  -T TLS-ECDHE-RSA-WITH-AES-256-GCM-SHA384 \
  -f 0 \
  -e "mssfix 0" \
  -e "txqueuelen 1000" \
  -e "management 0.0.0.0 5555" \
  -e "tls-version-min 1.2"


# Init PKI
docker-compose run --rm openvpn ovpn_initpki 
```
First command starts a basic configuration for the VPN, second starts the PKI, you'll be prompted for DN use the same as above (MYVPN.PUBLIC.IP.OR.DOMAIN) and also a password, make sure to use a safe one and save it somewhere safe!!

After that, it's suggested to run
```
sudo chown -R $(whoami): ./openvpn-data
```
Since the owner of this dir is root by default. 


## Extra notes on the setup
If you need help understanding the options of the first command, you can run `docker-compose run --rm openvpn ovpn_genconfig help` and it will show all the available options. However, be careful since the script will just ignore options that are configured wrong.
Remember to change **MYVPN.PUBLIC.IP.OR.DOMAIN** to your public IP or DNS name!

## Start docker-compose

Before starting docker-compose make sure the docker-compose.yml settings **OPENVPNMONITOR_DEFAULT_SITE** and **OPENVPNMONITOR_SITES_0_NAME** are adjusted to your needs. Also, the logo.png will be shown at the top level of the OpenVPN monitor page, you can change that to your needs as well

```
docker-compose up -d
```

## Generate, revoke and list client configuration

Generate certificates

```
# With passphrase
docker-compose run --rm openvpn easyrsa build-client-full CLIENT-NAME
# Without passphrase
docker-compose run --rm openvpn easyrsa build-client-full CLIENT-NAME nopass
```

Generate ovpn format file (to import to ovpn clients)

```
docker-compose run --rm openvpn ovpn_getclient CLIENT-NAME > CLIENT-NAME.ovpn
```

Revoke certificates

```
# Keep the corresponding crt, key and req files.
docker-compose run --rm openvpn ovpn_revokeclient CLIENT-NAME
# Remove the corresponding crt, key and req files.
docker-compose run --rm openvpn ovpn_revokeclient CLIENT-NAME remove
```

List clients

```
docker-compose run --rm openvpn ovpn_listclients
```

## References

* [Docker openvpn](https://github.com/kylemanna/docker-openvpn)
* [Docker openvpn monitor](https://github.com/ruimarinho/docker-openvpn-monitor)
* [OpenVPN speed optimization](https://community.openvpn.net/openvpn/wiki/Gigabit_Networks_Linux)
* [OpenVPN Hardening](https://community.openvpn.net/openvpn/wiki/Hardening)
* [Tunnelblick download page](https://tunnelblick.net/downloads.html)
* [UFW and docker](https://www.mkubaczyk.com/2017/09/05/force-docker-not-bypass-ufw-rules-ubuntu-16-04/)