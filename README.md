# Proof Of Concept Clients Linux, road map

Le but de ce dépôt Git est le suivant :

1. D'abord tenter d'implémenter l'intégration d'un client Linux
   *à l'aide d'Ansible*, le tout dans un environnement SambaÉdu4
   et via une commande manuelle lancée sur le serveur SambaÉdu
   (du genre `ansible -i "$CLIENT_IP," /etc/ansible/linuxclient.yaml`).
2. Puis ensuite d'évaluer la pertinence du choix de la solution
   Ansible dans l'intégration et le management des clients Linux
   dans SambaÉdu4.
3. Et c'est tout. :)

*Si* l'option « Ansible » est retenue, son intégration dans l'écosystème
SambaÉdu, son packaging, son automatisation en installations PXE etc.
etc. seront implémentés dans un deuxième temps (à travers un ou des
nouveaux dépôts Git).


# Contraintes sur les uid/gid des comptes du domaine

Avec un domaine Samba4 en mode AD, sur les clients Linux, il
faut _obligatoirement_ établir une plage des `uidNumber` et
`gidNumber` des comptes et groupes de l'annuaire AD qui sont
transposés en comptes et groupes Unix sur les client Linux.
C'est une plage écrite en dur dans la configuration Samba de
chaque client Linux. Par exemple si cette plage va 5 000 à 9
000 000, tout compte/groupe de l'annuaire AD possédant un
`uidNumber`/`gidNumber` en dehors de cette plage sera tout
simplement « ignoré » par le client Linux et donc non
exploitable sur le client Linux.

Proposition :

* Tout compte de l'annuaire AD exploitable sur un client Linux
  devra avoir un `uidNumber` entre 5 000 et 9 000 000.
* Tout groupe de l'annuaire AD exploitable sur un client Linux
  devra avoir un `gidNumber` entre 5 000 et 9 000 000.


# Bonnes pratiques sur les rôles Ansible

* Les variables de rôle devront être nommées suivant ce motif :
  `role_<nom-du-rôle>_xxx`. Par exemple, un nom de variable
  correct pour le rôle `ntp` est `role_ntp_servers`.

* Toutes les variables de rôle devront être mentionnées dans le
  fichier `roles/<nom-du-rôle>/defaults/main.yaml` soit avec
  une valeur par défaut générique quand c'est pertinent et soit
  de manière commentée quand il n'y a pas de valeur par défaut
  pertinente. Chaque variable, commentée ou non, devrait être
  accompagnée d'une description (en anglais please :)) de sorte
  que **le fichier `roles/<nom-du-rôle>/default/main.yalm` sera
  le mode d'emploi du rôle**.

* Pour chaque rôle, placer une task initiale qui teste la version
  de la distribution pour voir si elle matche avec les versions de
  distributions supportées par le rôle.

* Si jamais une task lance une commande (via les modules
  Ansible `shell` ou `command`) et que cette commande est amenée
  à contenir des données sensibles (comme typiquement un mot
  de passe), mettre impérativement `no_log: True` dans la task
  afin que la commande (avec son mot de passe etc.) ne soit
  pas logguée en clair dans le fichier `syslog` des machines
  cibles.

# Comment monter un partage avec Kerberos

Une fois connecté sur un client Linux, un compte toto du domaine
possède alors un ticket Kerberos dans `/tmp` qui est de la forme :

```sh
# Où 14001 est l'uidNumber du compte toto.
-rw------- toto domusers /tmp/krb5cc_14001_I1H5wf
```

Sous Stretch comme sous Xenial, on peut monter un partage
avec les droits de toto en utilisant le ticket Kerberos de
toto comme ceci :

```sh
USER='toto'

# On récupère le nom de fichier du ticket Kerberos de toto.
KRB5CCNAME=$(find /tmp/ -maxdepth 1 -mindepth 1 -type f -name 'krb5cc_*' -user "$USER")

KRB5CCNAME="$KRB5CCNAME" mount.cifs //se4.domain.tld/partage /mnt/docs/ \
    -o username="$USER",domain=DOMAIN.TLD,sec=krb5i,cruid="$USER"
```


# TODO

* Reprendre de zéro le script de logon.
* Questions : 

sur le se4, la partage netlogon-linux (monter en phase d'initialisation en lecture seule pour synchroniser le skel du se4 en local 
sur les clients linux) pourra-t-il fonctionner avec kerberos, sachant qu'aucun compte utilisateur AD ne s'est encore loggué sur le client en phase 
d'initialisation ? Autrement dit, ne faut-il pas utiliser ntlmv2 pour faire ce montage (uniquement) et donc rajouter :
```sh
raw NTLMv2 auth yes 
```
... au partage [netlogon-linux] du se4 afin de pouvoir faire le montage de netlogon-linux avec sec=ntlmv2 ?

Où faire la synchronisation du skel distant sur le client linux ? Sur se3, c'est au tout début de l'intégration que cela se faisait et 
l'intégration s'arrêtait si la synchro n'avait pas pu être faite.

* Sinon, si l'on reste sur la structure actuelle (synchronisation d'un skel distant en local), on pourrait repartir du script de logon actuel et 
y apporter quelques modifications pour qu'il soit fonctionnel sur se4.

Supprimer les lignes 32 à 60 et ne garder que la commande suivante (car le script de logon ne peut s'éxécuter qu'en local) :
```sh
exec 1> "$REP_LOG_LOCAL/1.initialisation.log" 2>&1 
```

Ligne 23 : définir le chemin local /etc/se3 comme variable ansible :
```sh
REP_SE3_LOCAL = {{ role_dm_local_skel_path  }}
```

Ligne 1430 : utiliser des variables Ansible à la place de celles du paquet se3 :
```sh
SE3= {{ role_dm_ip }}
BASE_DN= {{ role_dm_base_dn }}
SERVEUR_NTP= {{ role_dm_ntp_server }}
```

Ligne 1434 : définir la variable CHEMIN_PARTAGE_NETLOGON avec une variable ansible et en utilisant fqdn du se4 à la place de son IP
```sh
CHEMIN_PARTAGE_NETLOGON="//{{ role_dm_fqdn }}/$NOM_PARTAGE_NETLOGON"
```
Ligne 1447 : supprimer l’option sec=ntlmv2 de la variable OPTIONS_MOUNT_CIFS_BASE car le montage d'un partage Samba sur se4 devra pouvoir 
se faire en utilisant kerberos aussi.
```sh
OPTIONS_MOUNT_CIFS_BASE="nobrl,serverino,iocharset=utf8"
```

Ligne 505 : rajouter l'option sec=ntlmv2 (ou sec=krb5 mais je ne pense pas que cela fonctionnera ...) à la commande de montage de netlogon 
en phase d'initialisation 
```sh
mount -t cifs "$CHEMIN_PARTAGE_NETLOGON" "$REP_NETLOGON" -o ro,guest,sec=ntlmv2,"$OPTIONS_MOUNT_CIFS_BASE"
```

Ligne 602 et ailleurs ... : utilisation de la commande ldapsearch pour récupérer les groupes et utilisateurs de l'AD du se4. 
Y a-t-il un moyen plus simple en AD ?
Sinon, cette commande nécessite l'installation du paquet ldap-utils sur le client linux -> à rajouter dans une task du role dm ?

Ligne 556 : réécrire la fonction monter_partage_cifs() avec kerberos :
```sh
local KRB5CCNAME

# On cherche d'abord les credentials de l'utilisateur
KRB5CCNAME=$(find /tmp/ -maxdepth 1 -mindepth 1 -type f -name 'krb5cc_*' -user "$LOGIN")
if [ "$KRB5CCNAME"='' ]
then
	# au moment de l'execution de ces lignes de code, pam_krb5.so n'a pas fini de mettre en cache les credentials kerberos ...
	# mais ces derniers sont temporaiment dispo en root au format krb5cc_pam_****
	KRB5CCNAME=$(find /tmp/ -maxdepth 1 -mindepth 1 -type f -name 'krb5cc_pam_*' -user root)

	if [ "$KRB5CCNAME"='' ]
	then
		# Impossible récupérer les credentials kerberos de l'utilisateur:
			afficher_fenetre_erreur "Problème" \
            "Erreur lors de l'appel de la fonction \"monter_partage\"." \
            "Impossible de monter le partage \"$partage\"."
		return 4
	fi

	# On met temporairement l'utilisateur propriétaire du fichier pam
	chown "$LOGIN:" "$KRB5CCNAME"

	timeout "--signal=SIGTERM" 10s KRB5CCNAME="$KRB5CCNAME" mount.cifs "$1" "$2" \
		-o username="$LOGIN",domain={{ ansible_domain }},sec=krb5i,cruid="$LOGIN","$options"

	# On remet root propriétaire
	chown "root:root" "$KRB5CCNAME"	
else
	timeout "--signal=SIGTERM" 10s KRB5CCNAME="$KRB5CCNAME" mount.cifs "$1" "$2" \
		-o username="$LOGIN",domain={{ ansible_domain }},sec=krb5i,cruid="$LOGIN","$options"
fi
```

Ligne 1005 et 1006: faire le montage avec la nouvelle fonction kerberos  
monter_partage_cifs "$partage" "$point_de_montage" # port=139 faut-il rajouter cette option sur se4 ?

Supprimer tout ce qui concerne les credentials du paquet libpam-script à savoir la ligne 1458, ligne 1653 à 1685 et ligne 1753 à 1765
