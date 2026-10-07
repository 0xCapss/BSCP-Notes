---
tags: [bscp, server-side, ssrf]
niveau: practitioner
statut: à faire
---
# SSRF avec filtre d'entrée basé sur une liste noire

## En bref
- Certaines applications bloquent des noms d'hôtes comme `127.0.0.1` et `localhost`, ou des chemins sensibles comme `/admin`.
- Ces filtres sont presque toujours contournables, car ils comparent une chaîne et non l'adresse réelle visée.
- Note parente : [[SSRF]]

## Comment détecter
- Envoyer `http://127.0.0.1/` : une réponse d'erreur spécifique (blocage, `403`, message de sécurité) distingue un filtre applicatif d'une simple absence de service.
- Tester séparément l'hôte et le chemin pour savoir lequel des deux est bloqué.

## Comment exploiter (principe)
- Utiliser une autre représentation de l'adresse `127.0.0.1` :
	- Décimale : `2130706433`
	- Octale : `017700000001`
	- Abrégée : `127.1`
	- Hexadécimale : `0x7f000001`
	- IPv6 : `[::1]`
- Enregistrer un nom de domaine qui pointe vers `127.0.0.1` (par exemple `spoofed.burpcollaborator.net`).
- Masquer les chaînes bloquées par encodage d'URL ou en variant la casse (`ADMIN`, `%61dmin`).
- Fournir une URL contrôlée par l'attaquant qui redirige vers la cible, en essayant différents codes de redirection (`301`, `302`, `303`, `307`) et différents protocoles. Passer d'une URL `http:` à `https:` lors de la redirection a permis de contourner certains filtres.
- Si l'encodage simple est refusé, tester le double encodage : `%2561` décodé une première fois donne `%61`, puis `a`.

## Pièges et points d'attention BSCP
- Un filtre peut combiner plusieurs défenses : l'énoncé du lab le signale (« two weak anti-SSRF defenses »). Contourner l'hôte, puis le chemin, séparément.
- Bien vérifier que la représentation de l'adresse choisie est acceptée par le client HTTP du serveur, sinon la requête échoue même sans filtre.

## Labs PortSwigger
- [[SSRF#Lab 3 - SSRF avec filtre d'entrée basé sur une liste noire]] (Practitioner)

## Liens
- [[SSRF]]
- [[SSRF-filtre-liste-blanche]]
- [[SSRF-redirection-ouverte]]
