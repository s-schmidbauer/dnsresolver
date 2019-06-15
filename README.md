## basic system setup
echo "https://cdn.openbsd.org/pub/OpenBSD" > /etc/installurl
echo "permit keepenv nopass :wheel" > /etc/doas.conf
echo "inet alias 78.141.222.114 255.255.255.255" >> /etc/hostname.vio0
doas sh /etc/netstart vio0
doas fw_update
doas syspatch
doas pkg_add -u

### setup a user
adduser stefans
passwd stefans
echo "ecdsa-sha2-nistp521 AAAA.... " >> /home/stefans/.ssh/authorized_keys

### configure cron
see https://github.com/s-schmidbauer/wurstbot-dns/blob/master/crontab

#pf config
using existing IP blocklists collected by jupiter
doas ftp https://github.com/s-schmidbauer/wurstbot-dns/blob/master/pf.conf -o /etc/pf.conf
doas ftp https://github.com/s-schmidbauer/jupiter/blob/master/pf.mofos -o /etc/pf.mofos
doas ftp https://github.com/s-schmidbauer/jupiter/blob/master/pf.webmofos -o /etc/pf.webmofos
doas pfctl -nf /etc/pf.conf && doas pfctl -f /etc/pf.conf

## sshd config
doas ftp https://github.com/s-schmidbauer/wurstbot-dns/blob/master/sshd_config -o /etc/ssh/sshd_config
doas rcctl reload sshd

## prepare ad block lists
doas pkg_add curl
doas ftp https://github.com/s-schmidbauer/jupiter/blob/master/adblocks.sh -o /usr/local/bin/adblocks.sh
doas chmod +x /usr/local/bin/adblocks.sh
cd /tmp; adblocks.sh > adblocks.conf
doas cp adblocks.conf /var/unbound/etc/adblocks.conf

## tor
doas pkg_add tor
doas ftp https://github.com/s-schmidbauer/wurstbot-dns/blob/master/torrc -o /etc/tor/torrc
doas rcctl start tor
doas rcctl enable tor

## dnscrypt-proxy 
doas pkg_add dnscrypt-proxy
ftp https://github.com/s-schmidbauer/wurstbot-dns/blob/master/dnscrypt-proxy.toml -o /etc/dnscrypt-proxy.toml
doas rcctl set dnscrypt_proxy flags " -config /etc/dnscrypt-proxy.toml"
doas rcctl start dnscrypt_proxy
doas rcctl enable dnscrypt_proxy

## unbound 
ftp https://www.internic.net/domain/named.root -O /var/unbound/etc/root.hints
ftp https://github.com/s-schmidbauer/jupiter/blob/master/unbound.conf -o /var/unbound/etc/unbound.conf
doas chown _unbound: /var/unbound/etc/root.hints
doas unbound-anchor
doas rcctl start unbound
doas rcctl enable unbound

> warning! the config check fails with an out of memory error!
> the limits defined in /etc/login.conf only apply to processes started with rc.d
