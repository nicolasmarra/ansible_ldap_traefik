# Création d'un serveur LDAP 

Le but est de créer un serveur LDAP via ansible et à l'aide d'une image de conteneur et du role ansible décrit sur le sujet. De plus, le moteur de template d'ansible doit être utilisé et les variables décrites sur le sujet. 

## - Utilisation de Docker pour créer un serveur LDAP 

Avant de toucher à ansible, j'ai commencé par créer ce serveur LDAP en démarrant un conteneur avec l'image bitnami/openldap, en définissant les variables d'environnement pour le nom de l'administrateur, pour le mot de passe et la racine LDAP. 

```bash
docker run --name openldap -p 1389:1389 -e LDAP_ADMIN_USERNAME=admin -e LDAP_ADMIN_PASSWORD=tprli -e LDAP_ROOT="dc=ansible,dc=fr" -d  bitnami/openldap:latest
```

Ensuite, j'ai ajouté le fichier LDIF au serveur que je viens de créer afin d'importer les entrées LDAP de ce fichier : 

```bash
ldapadd -x -H ldap://localhost:1389 -D "cn=admin,dc=ansible,dc=fr" -w tprli -f fichier.ldif
```

Je vérifie que les utilisateurs ont bien été importés (ajoutés) au serveur LDAP en utilisant la commande ldapsearch: 

```bash
ldapsearch -x -H ldap://localhost:1389 -D "cn=admin,dc=ansible,dc=fr" -w tprli -b "dc=ansible,dc=fr"
 ...
 # search result
search: 2
result: 0 Success

# numResponses: 26
# numEntries: 25
 
```

La commande montre que plusieurs entrées ont été importées au serveur LDAP. 


## - Utilisation de Docker Compose pour créer un serveur LDAP

Le même conteneur LDAP a été créé via Docker Compose. Pour ce faire, j'ai créé le fichier `docker-compose.yml` suivant :

```yml 
services: 
  openldap: 
    image: bitnami/openldap:latest
    container_name: openldap
    ports:
      - '1389:1389'
    environment:
      LDAP_ADMIN_USERNAME: admin
      LDAP_ADMIN_PASSWORD: tprli
      LDAP_ROOT: dc=ansible,dc=fr
    volumes:
      - /tmp/openldap_data:/bitnami/openldap
    networks:
      - openldap_net
    user: root

networks:
  openldap_net:
```

Ensuite, j'ai utilisé la commande suivante pour démarrer le conteneur avec docker compose en arrière plan  :

```bash
docker-compose up -d
NAME       IMAGE                     COMMAND                  SERVICE    CREATED              STATUS              PORTS
openldap   bitnami/openldap:latest   "/opt/bitnami/script…"   openldap   About a minute ago   Up About a minute   0.0.0.0:1389->1389/tcp, 1636/tcp
```

Pour vérifier que le conteneur a été démarré, j'utilise la commande suivante : 

```bash
docker-compose ps
```
Ensuite, je vérifie les logs du conteneur LDAP et je constate donc que les données sont persistentes comme demandé sur le sujet : 

```bash
docker-compose logs openldap
openldap  |  07:22:55.87 INFO  ==> ** Starting LDAP setup **
openldap  |  07:22:55.92 INFO  ==> Validating settings in LDAP_* env vars
openldap  |  07:22:56.02 INFO  ==> Initializing OpenLDAP...
openldap  |  07:22:56.04 INFO  ==> Using persisted data
openldap  |  07:22:56.04 INFO  ==> ** LDAP setup finished! **
openldap  |
openldap  |  07:22:56.08 INFO  ==> ** Starting slapd **
openldap  | 67481a50.0556c82d 0x7fb87d7c0740 @(#) $OpenLDAP: slapd 2.6.9 (Nov 26 2024 23:27:17) $
openldap  |     @a5ab9e770e43:/bitnami/blacksmith-sandox/openldap-2.6.9/servers/slapd
openldap  | 67481a50.0879f9f9 0x7fb87d7c0740 slapd starting
```

J'importe les utilisateurs au serveur LDAP depuis le fichier LDIF : 

```bash
ldapadd -x -H ldap://localhost:1389 -D "cn=admin,dc=ansible,dc=fr" -w tprli -f fichier.ldif
```

Ensuite, je vérifie via ldpasearch que les utilisateurs ont été importés : 

```bash
ldapsearch -x -H ldap://localhost:1389 -D "cn=admin,dc=ansible,dc=fr" -w tprli -b "dc=ansible,dc=fr"
...
# search result
search: 2
result: 0 Success

# numResponses: 26
# numEntries: 25
```

Avant d'avancer de générer le docker compose à l'aide d'ansible, je vérifie la persistance de données de mon conteneur, j'ai rédemarré donc le conteneur, avec les commandes suivantes :

```bash
docker-compose down
docker-compose up -d
```

J'ai relancé la commande ldapsearch : 

```bash
ldapsearch -x -H ldap://localhost:1389 -D "cn=admin,dc=ansible,dc=fr" -w tprli -b "dc=ansible,dc=fr"

...

# search result
search: 2
result: 0 Success

# numResponses: 26
# numEntries: 25
```

La persistance des données est confirmée car toutes les entrées y sont toujours, cela est possible grâce au volume monté qui permet de conserver les données entre les redémerrages du conteneur.


## - Utilisation d'Ansible pour générer le docker compose (conteneur avec LDAP)

Pour générer le docker compose à l'aide d'Ansible, d'abord j'ai créé le fichier `inventory.yml`, dans ce fichier j'ai défini les hôtes et spécifié qu'Ansible doit se connecter localement sur localhost : 

```yml
all:
  children:
    test:
      hosts:
        localhost:
          ansible_connection: local Création du playbook Ansible
```

Ensuite, j'ai créé le playbook Ansible qui déploiera le serveur OpenLDAP sur la machine locale. Ce playbook utilisera le rôle openldap qui contient les configuration spécifiques à OpenLDAP, ainsi que le fichier LDIF et docker-compose.yml qui sera généré via Ansible. 


```yaml
- name: Déployer le serveur OpenLDAP
  hosts: test
  become: yes
  roles:
    - openldap
```

Le playbook appelle ensuite ce rôle qui demarrera le serveur LDAP dans un conteneur Docker via Docker Compose. Les variables d'environnements ont été définies aussi dans le fichier `main.yml` qui se trouve dans `roles/openldap/vars`

[Cliquez ici pour voir ce fichier](roles/openldap/vars/main.yaml)

```yaml
openldap_home: "/tmp/openldap_data"
openldap_admin_username: "admin"
openldap_admin_password: "tprli"
openldap_root: "dc=ansible,dc=fr"
```

Le template `docker-compose.j2` qui génère le fichier docker-compose.yml à partir des variables définies dans le fichie `main.yml`. 

[Cliquez ici pour voir ce fichier](roles/openldap/templates/docker-compose.j2)
```j2
services:
  openldap:
    image: bitnami/openldap:latest
    container_name: openldap
    ports:
      - '1389:1389'
    environment:
      LDAP_ADMIN_USERNAME: {{openldap_admin_username}}
      LDAP_ADMIN_PASSWORD: {{openldap_admin_password}}
      LDAP_ROOT: {{openldap_root}}
    volumes:
      - {{openldap_home}}:/bitnami/openldap
    networks:
      - openldap_net
    user: root

networks:
  openldap_net:
```

Ce modéle est ensuite utilisé pour générer le fichier docker-compose.yml.  

Les tâches ansible sont définies dans le fichier `main.yaml` qui se trouve dans `roles/openldap/tasks/main.yaml` : 

[Cliquez ici pour voir ce fichier](roles/openldap/tasks/main.yaml)

```yaml
- name: Créer le répertoire pour OpenLDAP
  file: 
    path: "roles/openldap/assets"
    state: directory
    mode: "0755"

- name: Créer le fichier docker-compose.yml
  template:
    src: docker-compose.j2
    dest: roles/openldap/assets/docker-compose.yml
    mode: "0755"

- name: Lancer le service docker-compose
  community.docker.docker_compose_v2:
    project_src: roles/openldap/assets


- name: Supprimer les anciennes entrées LDAP
  shell: ldapdelete -x -H ldap://localhost:1389 -D "cn={{openldap_admin_username}},{{openldap_root}}" -w {{openldap_admin_password}} -r "{{openldap_root}}"
  ignore_errors: yes

- name: Importer les données dans OpenLDAP
  shell: ldapadd -x -H ldap://localhost:1389 -D "cn={{openldap_admin_username}},{{openldap_root}}" -w {{openldap_admin_password}} -f roles/openldap/assets/fichier.ldif

- name: Afficher les données de l'instance OpenLDAP
  shell: ldapsearch -x -H ldap://localhost:1389 -D "cn={{openldap_admin_username}}, {{openldap_root}}" -w {{openldap_admin_password}} -b "{{openldap_root}}" 
  register: result

- name: Afficher les données récupérées
  debug:
    var: result.stdout

```

Ensuite, je lance mon playbook via la commande suivante :

```bash
 ansible-playbook -i inventory.yaml playbook_tp1.yaml
 ```

![alt text](/images/image.png)

On constante que la commande a bien marché et la sortie est aussi affiché. 

## - Questions 
 
- Pourquoi est-il pertinent d'attacher des conteneurs à plusieurs réseaux ?

Faciliter la communication entre les services et permettre à certains conteneurs d'intéragir entre différents réseaux, et aussi de maintenir une isolation entre eux. 


- Pourquoi certains ports ne sont pas ouvrable depuis votre utilisateur ?

Manque de privilèges : Les ports inférieurs à 1024 sont réservés au root.

# Creation d'une passerelle HTTP

Pour créer la passerelle HTTP avec Traefik via Ansible, j'ai commencé par créer un fichier `docker-compose.yaml` pour déployer Traefik dans un conteneur docker. [Cliquez ici pour voir ce fichier](roles/traefik/assets/docker-compose.yml)


Ensuite un rôle ansiblé a été mis en place pour gérer ce déploiement de manière automatisée. [Cliquez ici pour voir ce fichier](roles/traefik/tasks/main.yml)


Ensuite, je lance mon playbook via la commande suivante pour vérifier que tout se passe bien : 

```bash
 ansible-playbook -i inventory.yaml playbook_tp1.yaml
 ```

 Le dashboard est disponible sur l'url suivante : `http://localhost:8080/dashboard/#/`

![alt text](/images/Capture%20d'écran%202024-11-28%20210541.png)


# Creation d'un application WEB


Afin de tester le bon fonctionnement de Traefikn, le deploiment d'un conteneur `traefik/whoami` a été préconisé sur le domaine `whoami.localhost`. 

J'ai donc créé un fichier `docker-compose.yaml` et une tâche ansible pour déployer ce conteneur. Le fichier `inventory.yaml` ainsi que le playbook ont été modifés pour cet effet. 

Après modification, on peut confirmer le bon fonctionnement de traefik sur le domaine `whoami.localhost` : 

![alt text](/images/Capture%20d'écran%202024-11-28%20210500.png)

## - Questions : 

Sur quel couches du modèle OSI peut agir Traefik ?

Traefik peut agir sur la couche 4, la couche transport (UDP ou TCP), et surtout sur la couche 7 du modèle OSI, la couche application (HTTP/HTTPS).   

Qu'est qu'une ingress (vous pouvez faire le lien avec une ingress Kubernetes), un middleware et un plugin Traefik ?

Une ingress est une ressource Kubernetes qui gère l'accès HTTP et HTTPS aux services à l'intérieur d'un cluster. Elle permet de définir des règles de routage pour accéder aux services internes du cluster. Un middleware est un composant qui modifie les requêtes HTTP avant qu'elles soint envoyées aux services cibles, tandis qu'un plugin traefik est une extension qui permet d'ajouter des fonctionnalités à Traefik. 


Quelles expressions d'ingress permettent de capter:
* mondomaine.com/api/* et mondomaine2.fr/api/*
* tondomaine.com/api/* 
?

On peut utiliser `PathPrefix(/api)` : afin de rediriger les requêtes pour mon mondomaine/api vers le service api. 

```yml
"traefik.http.routers.api.rule=Host(`mondomaine.com`) && PathPrefix(`/api/`) || Host(`mondomaine2.fr`) && PathPrefix(`/api/`)"

```

Quel peut être l'interet d'utiliser une passerelle HTTP/HTTPS ?

L'intérêt d'utiliser une passerelle HTTP/HTTPS est la sécurité (certificat, contrôle d'accès et authentification, gestion des API). 

# Utiisation des groupes

Dans cette partie, j'ai ajouté 2 services dans le fichier `docker-compose.yml` du rôle whoami, un service qui génère un *conteneur Apache* sur le domaine `mon-site.localhost` et sur le chemin /front*, et un service qui génère un *conteneur Nginx* sur le domaine `mon-site.localhost` et sur le chemin /front*. J'ai dû utilise un `middleware` afin de supprimer le prefix du front et de l'API. Une tâche a été ajouté dans ce fichier afin de déployer tout cela. [Cliquez ici pour voir ce fichier](/roles/whoami/tasks/main.yml) et [cliquez ici pour voir le docker compose](/roles/whoami/assets/docker-compose.yml). 

Le playbook a été modifié afin de lancer le rôle web. [Cliquez ici pour le voir](/playbook_tp1.yaml)

On peut voir sur le domaine précisé avant que le déploiement a bien fonctionné. 

![alt text](/images/Capture%20d'écran%202024-11-28%20215433.png)

![alt text](/images/Capture%20d'écran%202024-11-28%20215528.png)

# Sécurisation du site de test

Afin de mettre en place l'authentification basique, j'ai utilisé ce plugin (https://plugins.traefik.io/plugins/628c9eb7ffc0cd18356a979c/ldap-auth) pour n'autoriser que les utilisateur (employés) à accéder à l'ingress `whoami.localhost`. J'ai donc modifié le fichier `docker-compose.yaml` de traefik pour ajouter l'authentification LDAP. [Cliquez ici pour voir le fichier](/roles/traefik/assets/docker-compose.yml)

D'ailleurs, pour que l'authentification marche, j'avais dû modifier les réseaux de mes conteneurs, j'avais trouvé deux approches, la première était de mettre le réseau de mon conteneur LDAP dans mon conteneur traefik, car toutes les autres services web utilisaient ce réseau aussi, après avoir réflechi, j'ai décidé de n'utiliser qu'un seul réseau, celui de mon conteneur traefik, il a été aussi défini dans mon conteneur LDAP.


[!alt text](/images/Capture%20d'écran%202024-11-28%20222815.png)



Maintenant le domaine `whoami.localhost` demande de faire l'authentification avant d'afficher les informations et si l'on met un utilisateur existant et avec le mot de passe, on voit les informations. 

[!alt text](/images/Capture%20d'écran%202024-11-28%20222824.png)


Si les logs, on vérifie que l'utilisateur est valide.

[!alt text](/images/Capture%20d'écran%202024-11-28%20222900.png)



## - Questions : 


Quel élément peut utiliser une application WEB derrière cette passerelle pour déterminer quel employé à envoyer la requête HTTP ?

Une application web derrière cette passerelle comme on a vu sur le log, peut utiliser le plugin ldap_auth pour transmettre le nom de l'utilisateur authentifié au serveur web, ou l'utilisateur qui a effectué la demande d'authentification.


Comment ajouterez-vous le fameux cadenas vert à ce domaine (Vous pouvez expliquer plusieurs solutions) ?

Pour ajouter le fameux cadenas vert, on a plusieurs approches possibles : 


- Utilisation de Let's Encrypt pour obtenir des certificats SSL gratuits ou alors de certificat auto-signé, cette solution est moins sécurisé et peu recommandée.
- Configuration de la passerelle via traefik, on peut avoir un certificat HTTPS sur le domaine whoami.localhost.



Combien de réseaux docker utilisez-vous et pourquoi ?

Comme expliqué précédemment, j'utilise un seul réseau docker (traefik_net) pour mon architecture afin de simplifier la gestion et les connexions entre les services. 
Cependant, pour une raison de sécurité et afin d'isoler les serveurs web et le serveur ldap, je pouvais avoir deux réseaux de docker, un pour les serveurs web et un deuxième pour le serveur ldap.