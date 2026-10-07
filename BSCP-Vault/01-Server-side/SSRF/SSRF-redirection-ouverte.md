---
tags: [bscp, server-side, ssrf]
niveau: practitioner
statut: à faire
---
# Contournement des filtres SSRF via une redirection ouverte

## En bref
- Condition : une application dont les URL sont autorisées contient une redirection ouverte, et l'API qui effectue la requête HTTP côté serveur suit les redirections.
- L'attaquant construit une URL qui satisfait le filtre mais aboutit à une requête redirigée vers la cible interne voulue.
- Note parente : [[SSRF]]

## Comment détecter
- Chercher dans l'application un paramètre qui alimente un en-tête `Location` (liens « produit suivant », « retour », `redirect`, `next`, `path`, `url`).
- Vérifier que la redirection mène bien vers un domaine arbitraire fourni dans le paramètre.
- Vérifier que le filtre SSRF refuse un hôte externe direct mais accepte l'URL du domaine local.

## Comment exploiter (principe)
- **Exemple**
	- L'URL `/product/nextProduct?currentProductId=6&path=http://evil-user.net` renvoie une redirection vers `http://evil-user.net`.
- **Exploitation**
	- L'attaquant envoie `stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin`.
- **Pourquoi cela marche ?**
	- L'application vérifie d'abord que l'URL de `stockApi` est sur un domaine autorisé, ce qui est le cas.
	- Elle interroge ensuite cette URL, ce qui déclenche la redirection ouverte.
	- Elle suit la redirection et envoie une requête vers l'URL interne choisie par l'attaquant.

## Pièges et points d'attention BSCP
- Dans le lab, `stockApi` peut contenir un chemin relatif (`/product/nextProduct?path=...`) : le filtre n'accepte que le serveur local.
- Si la redirection n'est pas suivie, cette technique est inutilisable : vérifier le comportement avec une URL externe contrôlée (Burp Collaborator).

## Labs PortSwigger
- [[SSRF#Lab 4 - SSRF avec contournement du filtre via une redirection ouverte]] (Practitioner)

## Liens
- [[SSRF]]
- [[SSRF-filtre-liste-blanche]]
- [[DOM-based]]
