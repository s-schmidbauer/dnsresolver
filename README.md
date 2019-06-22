## basic system setup
* echo "https://cdn.openbsd.org/pub/OpenBSD" > /etc/installurl
* echo "permit keepenv nopass :wheel" > /etc/doas.conf
* echo "inet alias 45.76.82.32 255.255.255.255" >> /etc/hostname.vio0
* doas sh /etc/netstart vio0
* doas fw_update
* doas syspatch
* doas pkg_add -u
* doas pkg_add git wget
* doas wget https://raw.githubusercontent.com/s-schmidbauer/wurstbot-dns/master/sysctl.conf -O /etc/sysctl.conf

### setup a user
* adduser -batch stefans wheel 'stefans' '$2b$08$wPQe2jmI3M4Rw1UpnRz8GOeNpW.LJLl5KcdQdZPXwNA94fBPBBwOq'
* echo "ecdsa-sha2-nistp521 AAAA.... " >> /home/stefans/.ssh/authorized_keys

### configure cron
see https://github.com/s-schmidbauer/wurstbot-dns/blob/master/crontab

## pf config
using existing IP blocklists collected by jupiter
* doas wget https://raw.githubusercontent.com/s-schmidbauer/wurstbot-dns/master/pf.conf -O /etc/pf.conf
* doas wget https://raw.githubusercontent.com/s-schmidbauer/jupiter/master/pf.mofos -O /etc/pf.mofos
* doas wget https://raw.githubusercontent.com/s-schmidbauer/jupiter/master/pf.webmofos -O /etc/pf.webmofos
* doas pfctl -nf /etc/pf.conf && doas pfctl -f /etc/pf.conf

## sshd config
* doas wget https://raw.githubusercontent.com/s-schmidbauer/wurstbot-dns/master/sshd_config -O /etc/ssh/sshd_config
* doas rcctl reload sshd

## prepare ad block lists
* doas pkg_add curl
* doas wget https://raw.githubusercontent.com/s-schmidbauer/wurstbot-dns/master/adblocks.sh -O /usr/local/bin/adblocks.sh
* doas chmod +x /usr/local/bin/adblocks.sh
* cd /tmp; adblocks.sh > adblocks.conf
* doas cp adblocks.conf /var/unbound/etc/adblocks.conf

## tor
* doas pkg_add tor
* doas wget https://raw.githubusercontent.com/s-schmidbauer/wurstbot-dns/master/torrc -O /etc/tor/torrc
* doas rcctl start tor
* doas rcctl enable tor

## dnscrypt-proxy 
* doas pkg_add dnscrypt-proxy
* wget https://raw.githubusercontent.com/s-schmidbauer/wurstbot-dns/master/dnscrypt-proxy.toml -O /etc/dnscrypt-proxy.toml
* doas rcctl enable dnscrypt_proxy
* doas rcctl set dnscrypt_proxy flags " -config /etc/dnscrypt-proxy.toml"
* doas rcctl start dnscrypt_proxy

## unbound 
* wget https://www.internic.net/domain/named.root -O /var/unbound/etc/root.hints
* wget https://raw.githubusercontent.com/s-schmidbauer/wurstbot-dns/master/unbound.conf -O /var/unbound/etc/unbound.conf
* doas chown _unbound: /var/unbound/etc/root.hints
* doas unbound-anchor
* doas rcctl enable unbound
* doas rcctl start unbound

> warning! the config check fails with an out of memory error!
> the limits defined in /etc/login.conf only apply to processes started with rc.d

## tipps
> when restarting unbound, dump the cache first and load it back in after the restart. Also useful for backup purposes
* doas unbound-control dump_cache > unbound.dump
* doas rcctl restart unbound ; cat unbound.dump | doas unbound-control load_cache
* rm unbound.dump
