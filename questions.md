## Création d'un serveur LDAP


- Pourquoi est-il pertinent d'attacher des conteneurs à plusieurs réseaux ?

Faciliter la communication entre les services et permettre à certains conteneurs d'intéragir entre différents réseaux, et aussi de maintenir une isolation entre eux. 


- Pourquoi certains ports ne sont pas ouvrable depuis votre utilisateur ?

Manque de privilèges : Les ports inférieurs à 1024 sont réservés au root.



## Création d'une application WEB

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


## Sécurisation du site de test

Quel élément peut utiliser une application WEB derrière cette passerelle pour déterminer quel employé à envoyer la requête HTTP ?

Une application web derrière cette passerelle comme on a vu sur le log, peut utiliser le plugin ldap_auth pour transmettre le nom de l'utilisateur authentifié au serveur web, ou l'utilisateur qui a effectué la demande d'authentification.


Comment ajouterez-vous le fameux cadenas vert à ce domaine (Vous pouvez expliquer plusieurs solutions) ?

Pour ajouter le fameux cadenas vert, on a plusieurs approches possibles : 


- Utilisation de Let's Encrypt pour obtenir des certificats SSL gratuits ou alors de certificat auto-signé, cette solution est moins sécurisé et peu recommandée.
- Configuration de la passerelle via traefik, on peut avoir un certificat HTTPS sur le domaine whoami.localhost.



Combien de réseaux docker utilisez-vous et pourquoi ?

Comme expliqué précédemment, j'utilise un seul réseau docker (traefik_net) pour mon architecture afin de simplifier la gestion et les connexions entre les services. 
Cependant, pour une raison de sécurité et afin d'isoler les serveurs web et le serveur ldap, je pouvais avoir deux réseaux de docker, un pour les serveurs web et un deuxième pour le serveur ldap.



