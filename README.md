
# Ansible et Passerelle Web avec Traefik

Ce projet utilise **Ansible** pour déployer un serveur **LDAP**, une **passerelle web** avec **Traefik**, ainsi que des serveurs web **Nginx** et **Apache**. 

## Prérequis

Pour exécuter ce projet, vous devez avoir les outils suivants installés sur votre machine :

- **Ansible**
- **LDAP**
- **Docker**

### Installation des dépendances :

1. Clonez ce repo :
   ```bash
   git clone https://git.unistra.fr/nmarra/tp_ansible
   cd tp_ansible
   ```

2. Installez les dépendances Python requises :
   ```bash
   pip install -r requirements.txt
   ```

3. Installez **Ansible** et **LDAP** si ce n'est pas déjà fait :
   ```bash
   sudo apt install ansible
   sudo apt install ldap-utils
   ```

## Exécution du projet

Une fois les prérequis installés, vous pouvez exécuter le playbook Ansible pour déployer l'infrastructure.

1. Exécutez le playbook Ansible avec la commande suivante :
   ```bash
   ansible-playbook -i inventory.yaml playbook_tp1.yaml
   ```

Cela va configurer et lancer tous les services nécessaires, y compris le serveur LDAP, les serveurs web Apache et Nginx, et la passerelle web Traefik.


### Fonctionnalités

- Authentification via **LDAP**.
- Utilisation de **Traefik** pour gérer les requêtes HTTP/HTTPS et assurer la sécurité du site via SSL.
- **Nginx** et **Apache** en tant que serveurs web servant de backends pour Traefik, avec des services comme **Whoami** exposés à travers ces serveurs.

