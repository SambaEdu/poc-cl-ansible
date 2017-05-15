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

Le script de logon sur se4 est à 99% repris de celui sur se3 : l'administration des clients linux est donc la même.
Les points importants à retenir :

* Le montage du partage netlogon-linux pendant la phase d'initialisation du script de logon ne peut pas être fait avec kerberos (car aucun ticket) :
  on garde donc sec=ntlmv2 et on ajoute l'option "raw NTLMv2 auth = yes" sur le partage netlogon-linux du se4 (ou alors on utilise sec=ntlmsspi ?).
  Rappel : le montage netlogon-linux est nécessaire pour faire la synchro du logon + le lancement des scripts unefois + maj éventuel du skel.
  
* les modifications majeures du logon sont : l'utilisation de kerberos pour faire le montage des partages samba et l'utilisation de ldbsearch
  à la place de ldapsearch pour faire des requêtes ldap au se4.
  
* logon_perso a été mis (pour l'instant) en commentaire dans le logon et seul le montage du partage [homes] de l'utilisateur est réalisé à l'ouverture de session.
  il faudra penser à décommenter et supprimer le montage temporaire.
  
* Le point délicat : pendant la phase d'ouverture du script de logon, le module pam_krb5.so n'a pas fini son travail sur le ticket kerberos de 
  l'utilisateur qui se loggue : le ticket kerberos de l'utilisateur loggué est disponible dans /tmp mais avec root comme propriétaire.

# TODO

Reste à faire:
* Tester que le fichier de conf smb_CIFSFS.conf du se3 est encore valable sur se4 ? 
  Il semble nécessaire de garder l'option raw NTLMv2 auth = yes sur le partage netlogon-linux.
  l'utilisation de l'option sec=ntlmsspi (à la place de ntlmv2) entraîne ensuite une erreur sur le montage des partages avec kerberos :
  
  mount error 11 = Resource temporarily unavailable
  Refer to the mount.cifs(8) manual page (e.g.man mount.cifs)
  
  
* Réécrire les deux fonctions afficher_liste_groupes() et afficher_liste_parcs() du script de logon mais avec la commande ldbsearch

* Point important pour le packaging : il faudra penser à "gérer" les balises d'échappement {{ raw }} ... {{ endraw }} du script de logon 
  (ou comprendre pourquoile format .j2 se plaint de certains caractères du script de logon.)

