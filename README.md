# Homelab-KubeCluster-Ansible

Ce code Ansible permet de provisionner un cluster Kubernetes k8s sur un ensemble de 7 Vms Proxmox crées au préalable avec [Terraform](https://gitlab.com/naim.assoum/homelab-provisionning). Le serveur de virtualisation Proxmox est installé sur un serveur Bare Metal Ovh.


L'architecture comprend :
* 1 `lb-node` pour le `load balancer` haproxy
* 3 `Worker-node` pour les `workers` du cluster Kubernetes
* 3 `Master-node` pour les `master` du cluster Kubernetes

Voici ci-dessous un schéma de l'architecture deployé :

![schema](schema-archi-cluster.png "schema")

Les roles Ansible utilisés sont les suivants :
* ``iptables` installe les rules Iptables nécessaires au bon fonctionnement du Cluster.
* `desinstall-k8s`, ce rôle permet de désinstaller k8s sur l'ensembles des vms.
* `setup-pre-k8s` qui installe les différentes dépendances pour la création du cluster et configure les hosts au niveau du noyau, réseau
* `packages-lb`qui installe les preréquis pour installer haproxy en tant que load-balancer.
* `haproxy` installe un conteneur docker `haproxy` qui sera utilisé en tant que loadbalancer pour les master nodes et l'api server.
* `k8s` crée le cluster. Le rôle initialise le cluster sur l'instance nommée `master-node-0` puis exécute les commandes join sur les autres machines master et sur les machines workers. Le rôle est idempotent, si le cluster est déjà initialisé ou si le service `kubelet` est up sur le node en question, il véfifiera juste que le cluster est dans un état correct en effectuant un curl sur l'url de healthcheck de l'`api server` .
* `desinstall-k8s`, ce rôle permet de désinstaller k8s sur l'ensembles des vms.
* `namespace`, ce rôle permet de créer les namespaces de base, après la création du cluster.
* `cluster-tools`, ce rôle installe tout les outils nécessaires au cluster via Helm et des manifest. Les outils installés sont l'ingress d'haproxy, flannel, metal-lb, etc...

Pour utiliser ce code, il faut disposer d'un inventaire dynamique pour viser les Vms Proxmox.

Voici la structure de l'inventaire utilisé dans ce repo:

```
---
plugin: community.general.proxmox
url: https://pasmonip:8006
user: "{{ lookup('ansible.builtin.ini', 'vault_user_proxmox_account', section='proxmox_account', file='inventories/vault.ini') }}"
password: "{{ lookup('ansible.builtin.ini', 'vault_password_proxmox_account', section='proxmox_account', file='inventories/vault.ini') }}"
validate_certs: false
want_facts: true
want_proxmox_nodes_ansible_host: true
cache: false
groups:
  master: "'master' in proxmox_name"
  worker: "'worker' in proxmox_name"
  lb: "'lb' in proxmox_name"
```

Ce fichier est utilisé en complement avec un vault.ini structuré de la manière suivante:

```
[proxmox_account]
vault_user_proxmox_account = pasmonuseridiot@pve
vault_password_proxmox_account = pasmonmotdepasseidiot
```

Ce fichier détient des données sensibles, il faut donc le vaulter avec ansible-vault. 

La configuration globale pour utiliser ce repo est défini dans un ansible.cfg à la racine avec cette structure suivante:

```
[defaults]
inventory = ./inventories/host-proxmox.yml
host_key_checking = False
forks = 10
roles_path = ./roles
vault_password_file=../vaults/vault_password_k8s_ansible
[inventory]
enable_plugins = host_list, script, auto, yaml, community.general.proxmox, file

[ssh_connection]
pipelining = True
```


On peut exécuter la commande suivante suivie du tag voulue selon le contexte :
```
ansible-playbook deploy.yml -t <tag>
```

Voici les contextes d'utilisation des tags globaux:
* `setup-pre-cluster` met en place tout les prérequis pour que l'installation du cluster et le cluster fonctionne
* `k8s` crée le cluster k8s
* `new-install` désinstalle le cluster puis relance une installation
* `cluster-tools` installe les outils du CLuster via Helm et des manifest


Après la création du cluster, il faut installer un `cni` afin que les différents nodes soient Ready, après cette installation, il faut redémarrer containerd sur l'ensemble du cluster.

La command ansible-console est parfaite pour cela:
```
ansible-console -l '!lb-node'
```