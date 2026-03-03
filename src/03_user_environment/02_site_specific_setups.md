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
https://zabbix.physik.uni-bonn.de,https://zabbix-test.physik.uni-bonn.de,https://web.physik.uni-bonn.de,https://web-dev.physik.uni-bonn.de,https://login.cern.ch,https://auth.cern.ch
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
You may also want to create the file `~/.config/oidc-agent/custom_parameters.config` with content:
```
[
  {
    "parameter": "claims_in_tokens",
    "value": "id_token token",
    "for_issuer": [
      "https://login.helmholtz.de/oauth2",
      "https://login-dev.helmholtz.de/oauth2"
    ],
    "request": [
      "auth_url"
    ]
  }
]
```
depending on your use case.

## Grid VOMS tools and Java JDK
Java JRE may be required by some GUIs, and JDK to build some:
```
yay -S jdk-openjdk
```
Also install VOMS tools and grid certificate authorities:
```
yay -S voms-clients igtf-trust-anchors
```
## CVMFS
Install package:
```
yay -S cvmfs
```
Edit `/etc/cvmfs/default.local` and add:
```
CVMFS_REPOSITORIES=cvmfs-config.cern.ch,atlas.cern.ch,atlas-nightlies.cern.ch,atlas-condb.cern.ch,belle.cern.ch,grid.cern.ch,sft.cern.ch,sft-nightlies.cern.ch,lhcb.cern.ch,lhcbdev.cern.ch,unpacked.cern.ch
CVMFS_QUOTA_LIMIT='2048'
CVMFS_HTTP_PROXY='http://somesquidproxy-i-can-se.example.com:3128;DIRECT'
```
Then, execute as `root` in a separate terminal:
```
. /etc/cvmfs/default.local
for A in $(echo $CVMFS_REPOSITORIES | tr ',' ' '); do
  echo $A;
  echo "$A /cvmfs/$A cvmfs noauto,x-systemd.automount,x-systemd.requires=network-online.target,x-systemd.idle-timeout=5min,x-systemd.requires-mounts-for=/cvmfs/cvmfs-config.cern.ch,_netdev 0 0" >> /etc/fstab;
  mkdir /cvmfs/$A;
done
```
You might want to organize `/etc/fstab` a bit after this, for example, add a headline `# CVMFS` and add some empty lines before and after the mounts. You should also remove the `x-systemd.requires-mounts-for=/cvmfs/cvmfs-config.cern.ch` from the config repository itself!
Afterwards, finalize and test things with:
```
systemctl daemon-reload
for A in /cvmfs/*; do mount $A; done
```
You may also want to regenerate the initrd just in case:;
```
mkinitcpio -P
```
and also reboot to get actual automounting working (this will enable the `.automount` units).

Finally, configure the user part. For this, create the file `~/.oh-my-zsh/custom/setupATLAS.zsh` with content:
```
setupATLAS() {
	if ls /cvmfs/atlas.cern.ch > /dev/null 2>&1; then
		echo "Setting up CVMFS environment..."
		export ATLAS_LOCAL_ROOT_BASE=/cvmfs/atlas.cern.ch/repo/ATLASLocalRootBase
		# ArchLinux, force container usage.
		export ALRB_containerSiteOnly=YES
		source ${ATLAS_LOCAL_ROOT_BASE}/user/atlasLocalSetup.sh "$@"
		return $?
	else
		echo " ERROR: CVMFS not available on this host"
		return 1
	fi
}
```
and the file `~/.oh-my-zsh/custom/setupBelle.zsh` with content:
```
setupBelle() {
	if ls /cvmfs/belle.cern.ch > /dev/null 2>&1; then
		echo "Setting up CVMFS environment..."
		source /cvmfs/belle.cern.ch/tools/b2setup
		return $?
	else
		echo " ERROR: CVMFS not available on this host"
		return 1
	fi
}
```
