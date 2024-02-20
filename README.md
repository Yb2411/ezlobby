# ezlobby


## Documentation Système VPN EZ-Lobby
### Architecture Système
#### Serveur CA (CA_server)
Le serveur d'authentification centralisé (CA_server) héberge plusieurs scripts essentiels et joue un rôle clé dans la gestion des certificats VPN, la génération de fichiers .ovpn, et l'interaction avec AWS pour le stockage et la gestion des configurations.

## Workflow de Souscription et Activation
### Étape 1 : Achat de Souscription
- Déclenchement via WooCommerce : Lorsqu'un utilisateur achète une souscription (ou un produit simple transformé en souscription via un snippet spécifique), le processus d'activation du VPN débute.
- Script create_client.sh : Ce script, situé sur /home/ec2-user, est exécuté automatiquement. Il effectue les opérations suivantes :
- Révocation de Certificat Existant : Vérifie si un certificat existant pour l'utilisateur est présent et le révoque le cas échéant. Cette étape assure qu'aucun conflit ou problème de sécurité ne survient avec les certificats précédents.
- Création de Nouveau Certificat : Génère un nouveau certificat VPN correspondant à la durée de la souscription. Ce certificat est conçu pour expirer automatiquement à la fin de la période de souscription.
### Étape 2 : Génération et Distribution des Fichiers OVPN
- Fichiers OVPN : Pour chaque destination VPN disponible (listée dans vpns_urls.txt), un fichier .ovpn est généré pour l'utilisateur.
- Envoi sur S3 : Les fichiers .ovpn générés sont ensuite envoyés sur un bucket AWS S3 via le script send_to_s3.py, facilitant ainsi leur distribution et accès par les utilisateurs.
### Étape 3 : Accès au VPN via "My VPN"
- Activation du Menu "My VPN" : Un snippet sur le site vérifie si la souscription de l'utilisateur est active. Si oui, le menu "My VPN" est activé, permettant à l'utilisateur d'accéder à ses fichiers VPN et informations de connexion.
### Étape 4 : Gestion des Tokens et Autorisations
- Insertion dans la Table tokens : Suite à la création d'une souscription ou à l'achat d'un produit simple converti en souscription, une entrée est insérée dans la table tokens de la base de données monitoring. Cette entrée contient le nom du client (utilisé comme identifiant unique) et la durée de l'abonnement.
- Champ revoked : Ce champ indique l'état actif (0) ou révoqué (1) de l'utilisateur. Il joue un rôle clé dans la gestion de l'accès aux services VPN et aux ressources associées.
### Étape 5 : Intégration avec AWS DynamoDB
- Création d'Entrée dans DynamoDB : Les utilisateurs autorisés, identifiés dans la table tokens, peuvent créer une entrée dans une table DynamoDB spécifique. Cette table gère la partie DNS pour l'utilisation du VPN sur les consoles, facilitant la configuration et le routage DNS personnalisé pour chaque utilisateur.
La table DynamoDB sur AWS joue un rôle central dans la gestion DNS, permettant une utilisation optimale du VPN sur console.

<img src="https://github.com/Yb2411/ezlobby/blob/main/smart_dns_arch.png" alt="smartdnsfonc" width="800" height="400">

#### Fonctionnement
- Chaque entrée dans DynamoDB contient le nom du client (ou token), son IP, et sa destination VPN choisie.
- Modification de la Table DynamoDB : À chaque changement, une fonction Lambda est déclenchée. Cette fonction exécute create_acls.py sur le serveur CA, générant deux fichiers ACLs :
- Un pour les DNS
- Un pour les serveurs proxy NGINX

<img src="https://github.com/Yb2411/ezlobby/blob/main/smart_dns_fonctionnel.png" alt="smartdnsfonc" width="600" height="900">

- ## Gestion des Expirations

La gestion des expirations est un processus crucial dans notre système VPN pour maintenir l'intégrité et la sécurité des sessions utilisateurs. Ce processus est orchestré principalement par le script `monitoring.py`, qui s'exécute dans une application Flask sur le port 5000, et est appuyé par une série de tâches cron pour la surveillance et la maintenance des sessions actives et expirées.

### Processus de Surveillance

1. **Surveillance des Logs** : Une tâche cron s'exécute toutes les deux minutes pour parcourir les fichiers log situés dans `/monitoring/*.log`. Cette tâche identifie les sessions actives. Lorsqu'une nouvelle session est détectée :
   - Si l'utilisateur n'existe pas déjà dans la base de données, il est inscrit avec les informations pertinentes, dont la durée de la session (`duration`).

2. **Vérification des Fichiers OpenVPN.log** : Parallèlement, chaque fois qu'un utilisateur est trouvé dans les logs OpenVPN, le système vérifie la présence de cet utilisateur dans la table `expirations`. Deux cas se présentent :
   - **Utilisateur Non Existant** : Si l'utilisateur n'est pas trouvé dans `expirations`, une nouvelle entrée est créée avec le `session start` défini à l'heure actuelle et le `session end` calculé en ajoutant la `duration` à l'heure actuelle.
   - **Vérification dans DynamoDB** : En outre, pour chaque utilisateur détecté, une vérification est effectuée contre la base DynamoDB pour s'assurer de sa présence dans la table `expirations` de MySQL. Si l'utilisateur n'est pas présent, il est ajouté; s'il existe déjà, aucune action supplémentaire n'est prise.

### Gestion des Expirations et Révocations

- **Tâche Cron pour les Expirations** : Une autre tâche cron tourne toutes les 30 minutes pour identifier les utilisateurs dont la session a expiré (c'est-à-dire lorsque `session end` est antérieur à l'heure actuelle). Ces sessions sont marquées pour révocation.
  
- **Vérification des Sessions Actives pour Utilisateurs Révoqués** : Simultanément, cette tâche vérifie également l'absence de sessions actives pour tout utilisateur marqué comme révoqué. En cas de détection d'une session active pour un utilisateur révoqué, une alerte est envoyée à `contact@ez-lobby.com` pour action.

- **Action de Révocation** : Lorsqu'une révocation est effectuée, l'enregistrement correspondant dans DynamoDB est supprimé pour retirer l'accès de l'utilisateur. De même, la colonne `revoked` dans la table `tokens` est mise à jour à 1, signalant ainsi la révocation de l'accès au site et aux téléchargements pour cet utilisateur.

Ce processus de gestion des expirations et révocations est essentiel pour s'assurer que seuls les utilisateurs actifs et autorisés accèdent au VPN, renforçant ainsi la sécurité et l'efficacité de notre système.

## Vérification DNS

### Processus de Vérification
- URL Principale : https://checkdns.ez-lobby.com/
- La plateforme utilise cette URL comme point de départ pour la vérification DNS. L'objectif est de s'assurer que les requêtes DNS des utilisateurs sont bien dirigées à travers notre infrastructure.

### Génération de Sous-domaine Aléatoire
- Pour éviter les problèmes de cache des navigateurs et assurer une vérification fiable, un sous-domaine aléatoire est généré pour chaque test. Ce sous-domaine suit le format *.checkdns.ez-lobby.com, où * représente la partie aléatoire générée pour la requête.
### Mécanisme de Vérification
#### Existence dans les Vues DNS 
- La zone "check" est configurée pour exister uniquement sur nos vues DNS. Ainsi, une réponse HTTP 200 à une requête vers ce sous-domaine aléatoire confirme que l'utilisateur est bien connecté via une vue DNS d'EZ-Lobby, bénéficiant ainsi de l'accès aux zones DNS spécifiques et à la redirection correcte.
#### Échec de la Résolution DNS : 
- Si la requête ne parvient pas à obtenir une réponse (c'est-à-dire aucun code HTTP 200), cela indique que l'utilisateur n'utilise pas nos services DNS. Cela est particulièrement utile pour le dépannage des configurations DNS sur les consoles où les vérifications traditionnelles via le jeu peuvent ne pas être disponibles.
### Utilité pour les Utilisateurs de Console
Cette méthode de vérification est spécialement conçue pour les joueurs sur console, offrant une solution simple pour vérifier leur configuration DNS sans avoir à quitter leur plateforme de jeu. Elle permet d'identifier rapidement et efficacement les problèmes de configuration DNS qui pourraient affecter l'expérience de jeu, la performance ou l'accès à des contenus géo-restreints.

## Backend

### Scripts et Fonctions
create_client.sh : Ce script est responsable de la révocation des certificats existants (si nécessaire) et de la création de nouveaux certificats correspondant à la durée de la souscription. Il génère ensuite des fichiers .ovpn pour chaque destination VPN listée dans vpns_urls.txt.
send_to_s3.py : Après la génération, ce script envoie les fichiers .ovpn vers un bucket S3 défini, permettant un accès facile par les clients.
Gestion des Souscriptions
Les souscriptions sont gérées via WooCommerce sur un site WordPress, où des snippets spécifiques détectent les achats et déclenchent l'exécution du create_client.sh sur le serveur CA pour initier la configuration VPN du client.

### Base de Données monitoring
Une base de données MySQL monitoring contient
- la table tokens, enregistrant chaque utilisateur avec son email (nom_client), la durée de son abonnement (duration), et un indicateur (revoked) pour suivre l'état de l'abonnement.
- la table expirations, qui contient la date de début de l'utilisation / souscription au vpn et inscrit la date de fin (utilisé par les cron pour revoker)
- la table session, qui enregistre les sessions vpn des users openvpn

### AWS DynamoDB et Fonction Lambda
La table DynamoDB stocke les informations des utilisateurs autorisés à créer une entrée pour la gestion DNS. Chaque modification dans cette table déclenche une fonction Lambda, qui exécute le script create_acls.py sur le CA_server pour générer des ACLs pour DNS et NGINX.

### Gestion DNS et Serveurs Proxy NGINX
Les serveurs DNS utilisent des vues pour diriger les requêtes vers les serveurs proxy NGINX appropriés, basés sur la destination VPN choisie par l'utilisateur. Les ACLs, générées à partir des données de DynamoDB, sont utilisées pour contrôler l'accès aux serveurs proxy.

### Automatisation avec Ansible
Des playbooks Ansible sont utilisés pour déployer et mettre à jour les configurations DNS et NGINX sur les serveurs correspondants:

- replication_dns.yml : Déploie les fichiers de configuration DNS et redémarre le service named.
- setup_nginx.yml : Installe et configure NGINX, y compris les fichiers de configuration spécifiques et redémarre NGINX.
- update_acl_and_restart_named.yml et update_nginx_acl.yml : Mettent à jour les ACLs pour DNS et NGINX et redémarrent les services concernés.

### Automatisation et Monitoring
Des tâches cron sont configurées pour :

- Extraire les logs OpenVPN.
- Vérifier les expirations et révoquer les accès si nécessaire.
- Assurer qu'aucune session n'est active pour un utilisateur révoqué.
- Envoyer des alertes par email en cas de problème.

Le endpoint ez-lobby.com/monitoring fournit des statistiques sur l'utilisation du VPN, incluant les connexions actives, la durée des sessions, et les destinations populaires.
