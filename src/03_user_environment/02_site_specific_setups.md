# Site-specific setups

## Kerberos 5 setup

Edit `/etc/krb5.conf` and make sure it contains the following (note `mdbook` sadly converts all tabs to spaces, you may want to use tabs here):
```
[libdefaults]
	default_realm = UNI-BONN.DE
	forwardable = true
	proxiable = true
	default_ccache_name = KEYRING:persistent:%{uid}
	ticket_lifetime = 1day
	renew_lifetime = 7days

[realms]
	CERN.CH = {
		default_domain = cern.ch
		kdc = cerndc.cern.ch
		admin_server = cerndc.cern.ch
		kpasswd_server = cerndc.cern.ch
	}
	UNI-BONN.DE = {
		kdc = kdc.uni-bonn.de
		kdc = kdc1.uni-bonn.de
		kdc = localhost:8987
		default_domain = uni-bonn.de
		admin_server = kdc.uni-bonn.de
		admin_server = kdc1.uni-bonn.de
		admin_server = localhost:8987
		#port 749 ?
	}
[domain_realm]
	cern.ch = CERN.CH
	.cern.ch = CERN.CH
	uni-bonn.de = UNI-BONN.DE
	.uni-bonn.de = UNI-BONN.DE
	rhrz.uni-bonn.de = UNI-BONN.DE
	.rhrz.uni-bonn.de = UNI-BONN.DE
	
[logging]
#       kdc = CONSOLE

[appdefaults]
login = {
	forwardable = true
	krb5_run_aklog = true
	krb5_get_tickets = true
	krb4_get_tickets = false
	krb4_convert = false
}
kinit = {
	forwardable = true
	proxiable = true
	krb5_run_aklog = true
}
```
You can of course leave existing domain configuration in.

Note that the `localhost` part here is for forwarding the KDC via `socat`. 

## Firefox

Set `network.negotiate-auth.trusted-uris` in `about:config` to:
```
https://zabbix.physik.uni-bonn.de,https://web.physik.uni-bonn.de,https://web-dev.physik.uni-bonn.de,https://login.cern.ch,https://auth.cern.ch
```

## OIDC Agent setup
Install the package:
```
yay -S oidc-agent
```
then enable this in `~/.zshrc` by adding:
```
eval `oidc-agent-service use` > /dev/null
```
