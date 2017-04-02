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



