# Changelog

## Version 2.2.0
 * feat: rôle pour créer les namespaces de base
 * feat: rôle pour installer via helm et le module kubernetes, les charts de base dans le cluster
 
## Version 2.1.0
  * Installation des dépendances de longhorn pour l'infra 
  * Rajout de la gestion des IpTables as Code avec Ansible
  
## Version 2.0.0
 * Passage d'un inventaire statique à un inventaire dynamique, changement de nom de group_vars et d'host sur le playbook `deploy.yml`
 
## Version 1.0.0
 * Création du repo Ansible permettant de créer, supprimer un cluster Kubernetes sur Proxmox VE
