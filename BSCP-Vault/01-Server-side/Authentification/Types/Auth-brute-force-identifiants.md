---
tags: [bscp, server-side, authentication]
niveau: apprentice
statut: à faire
---
# Authentification - brute force et énumération des identifiants

## En bref
- Attaque par essais automatisés de noms d'utilisateur et de mots de passe, avec Burp Intruder.
- L'efficacité dépend de la reconnaissance : une liste ciblée vaut mieux qu'une liste générique.
- Note parente : [[Authentication]]

## Comment détecter
- Comparer la réponse à un couple entièrement invalide et à un username valide avec un mauvais mot de passe : code de statut, message d'erreur, longueur, temps de réponse.
- Chercher les usernames divulgués : profils publics, e-mails dans les réponses HTTP, auteurs de commentaires.

## Comment exploiter (principe)
- Méthode d'essai/erreur automatisée avec des listes de noms d'utilisateur et de mots de passe potentiels, à l'aide d'un outil comme Burp Intruder.
- S'appuie sur une logique élémentaire ou des informations récupérées lors de la reconnaissance pour augmenter l'efficacité de l'attaque, plutôt que sur des listes génériques.

**Brute force des noms d'utilisateur**
- Les identifiants professionnels suivent souvent un schéma reconnaissable (`prenom.nom@compagnie.com`).
- Des comptes à privilèges élevés sont parfois créés avec des noms prévisibles comme `admin` ou `Administrator`.
- Toujours vérifier si le site divulgue publiquement des noms d'utilisateur potentiels : profils publics (même avec un contenu masqué, le nom affiché est parfois identique à l'identifiant de connexion), adresses e-mail visibles dans les réponses HTTP, y compris celles de comptes à privilèges élevés.

**Brute force des mots de passe**
- De nombreux sites imposent une politique de mot de passe à forte entropie (longueur minimale, combinaison majuscule/minuscule, caractères spéciaux).
- Le comportement humain introduit des failles malgré cela : les utilisateurs adaptent un mot de passe mémorisable pour respecter la politique plutôt que d'en choisir un vraiment aléatoire.
	- Exemple : si `mypassword` est refusé, l'utilisateur essaiera `Mypassword1!` ou `Myp4ssw0rd`.
	- Lors d'un changement de mot de passe imposé, les utilisateurs appliquent souvent une modification mineure : `Mypassword1!` devient `Mypassword1?` ou `Mypassword2!`.
- Cette prévisibilité permet de construire des listes de mots de passe candidats bien plus efficaces qu'une liste générique.

**Énumération des noms d'utilisateur par différence de comportement**
- Consiste à observer les changements de comportement du site pour déterminer si un username est valide, généralement sur la page de connexion (username valide + password invalide, comparé aux deux invalides).
- Réduit fortement le temps nécessaire pour un brute force complet : plus besoin de deviner username et password en même temps.
- Signaux à surveiller : code de statut différent, message d'erreur différent, temps de réponse anormal (un écart par rapport à la norme suggère un traitement différent en arrière-plan, par exemple un hachage du mot de passe qui ne s'exécute que si le username existe).

## Pièges et points d'attention BSCP
- Une différence subtile (point final manquant, espace en trop) se détecte avec une règle *Grep - Extract* sur le message d'erreur plutôt qu'à l'œil.
- Une différence de temps n'apparaît parfois qu'avec un mot de passe très long (100 caractères ou plus) : le hachage ne s'exécute que si le username existe.
- Ne pas se fier à un seul type de signal : codes, messages, longueurs et temps peuvent chacun être le bon indicateur.

## Labs PortSwigger
- [[Authentication#Lab 1 - Énumération de noms d'utilisateur via des réponses différentes]] (Apprentice)
- [[Authentication#Lab 2 - Énumération de noms d'utilisateur via des réponses subtilement différentes]] (Practitioner)
- [[Authentication#Lab 3 - Énumération de noms d'utilisateur via le temps de réponse]] (Practitioner)
- [[Authentication#Lab 5 - Énumération de noms d'utilisateur via le verrouillage de compte]] (Practitioner)

## Liens
- [[Authentication]]
- [[Auth-contournement-blocage]]
- [[Payloads-cheatsheet]]
