# Commandes pour intégrer une Ubuntu Xenial dans un domaine Samba 4

Il s'agit ici d'une intégration **de base**, avec juste la
possibilité d'ouvrir une session graphique via un compte du
domaine, rien de plus. Pas de gestion des montages, pas de
script de login, juste la mise à place de *l'authentification*
sur le display manager.

Voici le code shell qu'il faudra tenter de convertir en du code Ansible :

```sh
REAL='ATHOME.PRIV' # c'est nom de domaine du réseau Samba4.
WORKGROUP='ATHOME'
ADM_PWD='123456' # C'est le mot de passe du compte administrateur du domaine AD.

# Avec un AD, c'est juste impératif d'avoir des clients synchro avec le serveur Samba4.
apt-get install -y ntp

# Éventuellement modifier les serveurs NTP de référence pour
# requêter une machine du réseau local qui fait office de
# serveur de temps (le serveur Samba lui-même ou le serveur
# AMON par exemples).
vim /etc/ntp.conf && service ntp restart

# Ce démon ne sert à rien et peut mettre la pagaille, on le vire.
# Mauvaise idée au niveau des dépendances de paquets. On laisse.
# apt-get purge -y avahi-daemon

# Vérification à faire, normalement le fichier devrait être
# OK après l'installation (typiquement via la configuration
# DHCP).
#
# Au niveau de la configuration réseau, le serveur DNS doit
# être impérativement le serveur Samba4. On peut peut-être
# considérer que c'est d'office le cas (via le configuration
# DHCP du réseau).
vim /etc/resolv.conf

# Mise en place de Kerberos.
apt-get install -y krb5-user

cat >/etc/krb5.conf <<EOF
[libdefaults]
    default_realm    = $REAL
    dns_lookup_realm = false
    dns_lookup_kdc   = true
    ticket_lifetime  = 1d
    clockskew        = 300

EOF

# Mise en place de la configuration Samba et surtout Winbind.
apt-get install -y samba winbind samba-dsdb-modules

service smbd stop
service winbind stop
service nmbd stop

# On ne fait aucun partage sur cet hôte (les partages sont
# sur le serveur Samba4), donc on peut désactiver smbd et
# nmbd. En fait, c'est juste le daemon winbind qui nous
# intéresse ici.
systemctl disable nmbd.service
systemctl disable smbd.service

# Il faut partir de zéro, alors on supprime toutes les
# données internes de samba.
find /var/lib/samba/ '(' -name '*.tdb' -o -name '*.ldb' ')' -delete


cat >/etc/samba/smb.conf <<EOF
[global]
       security  = ADS
       workgroup = $WORKGROUP
       realm     = $REALV
       log file  = /var/log/samba/%m.log
       log level = 1

       # Default ID mapping configuration for local BUILTIN accounts
       # and groups on a domain member. The default (*) domain is
       # mandatory and:
       # - must not overlap with any domain ID mapping configuration!
       # - must use an read-write-enabled back end, such as tdb.
       #
       # Warning: the ranges of all backends must not overlap.
       #
       idmap config * : backend = tdb
       idmap config * : range   = 2000-2999

       # To retrieve ID mapping _from_ the domain ATHOME (on the DC).
       idmap config ATHOME : backend = ad
       idmap config ATHOME : range = 3000-50000
       idmap config ATHOME : schema_mode = rfc2307

       # To retrieve homedir, shell etc. from the AD.
       # But it will be:
       #
       #       idmap config ATHOME:unix_nss_info = yes
       #
       # On Samba 4.6 and later.
       #
       winbind nss info = rfc2307

       # To avoid "ATHOME\\user" in the output of "getent passwd"
       # and have just "user" (in the first field).
       winbind use default domain = yes

       winbind enum users         = yes
       winbind enum groups        = yes
       winbind separator          = /

       domain master   = no
       local master    = no
       disable netbios = no

EOF

# Pour que le système utilise winbind pour les résolutions
# des uid/gid du domaine (ie non locaux).
apt-get install -y libnss-winbind

# On édite ces deux lignes (et seulement ces deux lignes)
# pour avoir :
#
#   passwd:         compat winbind
#   group:          compat winbind
#
vim /etc/nsswitch.conf

# Moment fatidique de la jonction au domaine.
net ads join -U administrator%"$ADMIN_PWD" && echo OK

# On peut désormais faire repartir le service winbind.
# Normalement, à ce stade, on voit les comptes du domaine
# avec « getent -s winbind passwd ».
service winbind start


# Configuration de PAM afin que le display manager (Lightdm
# ici) puisse procéder à l'authentification via Kerberos.
apt-get install libpam-krb5

# On change la configuration de Lightdm pour qu'on puisse
# saisir manuellement le login.
cat >/etc/lightdm/lightdm.conf.d/custom.conf <<EOF
[SeatDefaults]
    greeter-show-manual-login = true
    greeter-hide-users        = true
    allow-guest               = false
EOF

service lightdm restart
```

Normalement, on doit alors pouvoir ouvrir une session
graphique avec le compte `toto` du domaine, il reste juste à
faire :

```sh
mkdir /home/toto
chown toto: -R /home/toto
```

puis se connecter sur Lightdm avec comme login juste `toto`.


Remarque : 
Cette intégration fonctionne également pour les clients lourds ltsp à l'exception de la commande net ads join [...]
qui ne doit pas être lancer dans le chroot sur le serveur ltsp.

Pour que ltsp sur se4 soit fonctionnel, il faut :

* Compléter le fichier /etc/lts.conf du chroot :
DNS_SERVER="IP_SE4"
SEARCH_DOMAIN="athome.priv"
HOSTNAME="ltsp"
TIMESERVER="IP_SE4"

* Régler correctement le timezone dans le chroot :
echo 'Europe/Paris' > /etc/timezone

* Modifier pam_mount afin que le montage des partages samba du se4 se fasse avec kerberos :
Le volume pam_mount devra être sous le format suivant :
		user="*"
		sgrp="lcs-users"
        fstype="cifs"
        server="$FQDN_SE4"
        path="homes/Docs"
        mountpoint="~/Docs sur le réseau"
		options="nobrl,serverino,iocharset=utf8,sec=krb5i,cruid=%(USERUID),user=%(DOMAIN_USER)"
		
* Faire démarrer un client lourd sur le réseau. Se connecter avec le compte enseignant, puis passer en root 
sur un émulateur de console et lancer la commande qui permet de joindre le client lourd au domaine se4 :

net ads join -U administrator%"$ADMIN_PWD"

* Copier ensuite tous les fichiers d'intégration créés sur le client lourd vers son chroot sur le serveur ltsp :
scp '/var/lib/samba/*.tdb' root@SERVEUR_LTSP:"/opt/ltsp/i386/var/lib/samba/'
scp '/var/lib/samba/*.ldb' root@SERVEUR_LTSP:"/opt/ltsp/i386/var/lib/samba/'

* Sur le serveur ltsp, reconstruire l'image squashfs (NBD est utilisé sur Stretch):
ltsp-update-image i386

* Redémarrer le client lourd et vérifier qu'il est possible de s'identifier avec un compte du domaine se4 
et que les partages réseau sont montés sur le bureau

* Dans le chroot, supprimer l'option HOSTNAME="ltsp" de /etc/lts.conf (sinon, tous les clients lourds en fonctionnement auront le même hostname "ltsp")




