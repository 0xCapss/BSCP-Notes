---
tags: [bscp, server-side, authentication]
niveau: apprentice
statut: en cours
---
# Authentication

## En bref
- Processus qui consiste à vérifier qu'un utilisateur est bien celui qu'il prétend être (souvent via un formulaire nom d'utilisateur/mot de passe).
- Une authentification vulnérable permet à un attaquant d'accéder à des données ou fonctionnalités sensibles : compromission totale de l'application si le compte visé a des privilèges élevés (ex: admin système), accès à des données normalement hors de portée même via un compte à bas privilège, et élargissement de la surface d'attaque via l'accès à de nouvelles pages/fonctions.

![](Authentication.png)

## Types et variantes
- Authentification par mot de passe (formulaire de connexion) : l'utilisateur prouve son identité par la connaissance d'un secret.
- Authentification HTTP (Basic Auth) : le navigateur envoie un token `base64(username:password)` dans l'en-tête `Authorization` à chaque requête.
- Authentification multi-facteurs (voir [[Multi-factor-authentication]]) : combine plusieurs des facteurs ci-dessous.
- Les 3 facteurs d'authentification possibles :
	- Facteur de connaissance : quelque chose que l'on connaît (mot de passe, réponse à une question de sécurité).
	- Facteur de possession : quelque chose que l'on possède (objet physique, token, téléphone).
	- Facteur inhérent : quelque chose qui nous est propre (données biométriques).
- Deux origines de vulnérabilité à distinguer : faiblesse inhérente au mécanisme lui-même (pas de protection contre le brute force) contre faille logique ou erreur d'implémentation qui permet de contourner totalement le mécanisme.

## Comment détecter
- Vérifier si l'application divulgue des noms d'utilisateur valides : pages de profil publiques, adresses e-mail visibles dans les réponses HTTP, schéma prévisible type `prenom.nom@entreprise.com`, comptes à privilèges élevés avec des noms devinables (`admin`, `administrator`).
- Sur le formulaire de connexion, comparer les réponses entre un couple username/password totalement invalide et un username valide avec un password invalide : code de statut, message d'erreur, longueur de réponse, temps de réponse. Toute différence signale une énumération de username possible.
- Tester si un verrouillage de compte existe, et si oui, quel est son déclencheur exact (nombre de tentatives par compte, par IP, ou par la combinaison des deux).
- Repérer les mécanismes d'authentification en plusieurs étapes (2FA) pour vérifier si l'état de session est déjà "connecté" avant la validation complète de la deuxième étape.
- Vérifier le format des en-têtes de la requête si le site utilise l'authentification HTTP Basic (`Authorization: Basic ...`) et si le protocole HSTS est en place.

## Comment exploiter (principe)
### Brute force des identifiants
- Méthode d'essai/erreur automatisée avec des listes de noms d'utilisateur et de mots de passe potentiels, à l'aide d'un outil comme Burp Intruder.
- S'appuie sur une logique élémentaire ou des informations récupérées lors de la reconnaissance pour augmenter l'efficacité de l'attaque, plutôt que sur des listes génériques.

### Brute force des noms d'utilisateur
- Les identifiants professionnels suivent souvent un schéma reconnaissable (`prenom.nom@compagnie.com`).
- Des comptes à privilèges élevés sont parfois créés avec des noms prévisibles comme `admin` ou `Administrator`.
- Toujours vérifier si le site divulgue publiquement des noms d'utilisateur potentiels : profils publics (même avec un contenu masqué, le nom affiché est parfois identique à l'identifiant de connexion), adresses e-mail visibles dans les réponses HTTP, y compris celles de comptes à privilèges élevés.

### Brute force des mots de passe
- De nombreux sites imposent une politique de mot de passe à forte entropie (longueur minimale, combinaison majuscule/minuscule, caractères spéciaux).
- Le comportement humain introduit des failles malgré cela : les utilisateurs adaptent un mot de passe mémorisable pour respecter la politique plutôt que d'en choisir un vraiment aléatoire.
	- Exemple : si `mypassword` est refusé, l'utilisateur essaiera `Mypassword1!` ou `Myp4ssw0rd`.
	- Lors d'un changement de mot de passe imposé, les utilisateurs appliquent souvent une modification mineure : `Mypassword1!` devient `Mypassword1?` ou `Mypassword2!`.
- Cette prévisibilité permet de construire des listes de mots de passe candidats bien plus efficaces qu'une liste générique.

### Énumération des noms d'utilisateur par différence de comportement
- Consiste à observer les changements de comportement du site pour déterminer si un username est valide, généralement sur la page de connexion (username valide + password invalide, comparé aux deux invalides).
- Réduit fortement le temps nécessaire pour un brute force complet : plus besoin de deviner username et password en même temps.
- Signaux à surveiller : code de statut différent, message d'erreur différent, temps de réponse anormal (un écart par rapport à la norme suggère un traitement différent en arrière-plan, par exemple un hachage du mot de passe qui ne s'exécute que si le username existe).

### Contournement d'un verrouillage de compte (password spraying)
- Établir une liste de noms d'utilisateur potentiellement valides et une liste très restreinte de mots de passe probables.
- Avec Burp Intruder, tester chaque mot de passe contre chaque username (attaque en grille) : il suffit qu'un seul utilisateur ait choisi l'un des mots de passe testés pour compromettre un compte, sans jamais dépasser le seuil de verrouillage par compte.
- Le verrouillage par compte ne protège pas non plus contre le credential stuffing (test d'un grand nombre de paires username/password déjà connues, issues de fuites d'autres sites), qui exploite la réutilisation de mots de passe entre sites.

### Contournement d'un blocage par IP
- Une IP peut être bloquée après un nombre trop élevé de tentatives, avec déblocage automatique après un délai, manuel par un administrateur, ou via un CAPTCHA résolu par l'utilisateur.
- Un attaquant peut manipuler son IP apparente (en-têtes `X-Forwarded-For`, `X-Real-IP`, etc.) pour contourner ce blocage si l'application fait confiance à un en-tête fourni par le client plutôt qu'à la connexion réelle (voir [[Payloads-cheatsheet]]).

### Authentification HTTP Basic
- Le client reçoit du serveur un token d'authentification qui est la concaténation `username:password` encodée en base64.
- Ce token est géré par le navigateur et ajouté à chaque requête dans l'en-tête `Authorization` :
`Authorization: Basic base64(username:password)`
- Cette méthode est peu sécurisée car :
	- Elle implique l'envoi répété des identifiants à chaque requête.
	- Sans HSTS, les identifiants risquent d'être interceptés lors d'une attaque man-in-the-middle.
	- Les implémentations ne prennent souvent pas en charge de protection contre le brute force.
	- Elle est particulièrement vulnérable aux exploits liés à la session, notamment le CSRF.

## Pièges et points d'attention BSCP
- Le verrouillage de compte ou d'IP n'est pas une protection absolue : inclure ses propres identifiants valides à intervalles réguliers dans la liste testée suffit souvent à passer sous le radar, ou révèle que le compteur se réinitialise après un login réussi.
- Toujours vérifier si le seuil de blocage est scopé par IP, par compte, ou par la combinaison des deux : ça change complètement la stratégie (rotation d'IP vs répartition sur plusieurs comptes).
- Une différence de temps de réponse peut être un signal d'énumération même quand les codes de statut et les messages sont identiques : ne pas se fier à un seul type de signal.
- Sur un mécanisme 2FA, vérifier si l'état de session est déjà "connecté" avant validation complète de la deuxième étape, et si le serveur revérifie bien cette étape avant d'afficher les pages protégées.

## Prévention
- Toujours renvoyer le même code de statut et le même message d'erreur générique, que le username ou le password soit invalide.
- Uniformiser le temps de réponse entre les cas valides et invalides (éviter qu'un traitement conditionnel, comme le hachage du mot de passe, ne s'exécute que si le username existe).
- Verrouiller le compte ciblé après un nombre élevé de tentatives infructueuses, et/ou bloquer l'IP distante en cas de fréquence de connexion anormale.
- Ne pas se reposer uniquement sur le blocage par IP pour la protection anti-brute-force : il est contournable par manipulation d'en-têtes si l'IP du client n'est pas déterminée de façon fiable côté serveur.
- Ne jamais utiliser l'authentification HTTP Basic seule pour protéger des ressources sensibles ; si utilisée, l'associer systématiquement à HSTS et à une protection anti-brute-force dédiée.

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
- 

## Mes notes
- Différence entre authentification et autorisation :
	- Authentification : processus qui consiste à vérifier qu'un utilisateur est bien celui qu'il prétend être.
	- Autorisation : consiste à vérifier si un utilisateur est autorisé à effectuer une action.

## Liens
- [[Access-control]]
- [[JWT-attacks]]
- [[OAuth]]
- [[Business-logic]]
- [[Multi-factor-authentication]]
- [[Payloads-cheatsheet]]
