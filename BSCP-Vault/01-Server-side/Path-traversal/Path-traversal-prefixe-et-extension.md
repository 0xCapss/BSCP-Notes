---
tags: [bscp, server-side, path-traversal]
niveau: practitioner
statut: à faire
---
# Path traversal - validation du préfixe et de l'extension

## En bref
- L'application impose une forme au chemin fourni : il doit commencer par un répertoire de base, ou se terminer par une extension attendue.
- Note parente : [[Path-traversal]]

## Comment exploiter (principe)
- **Validation du préfixe** : une application peut valider que le chemin fourni commence bien par un répertoire de base attendu (par exemple `/var/www/images/`) sans empêcher la remontée ensuite. On fournit un chemin qui satisfait ce préfixe puis remonte quand même :
	- `filename=/var/www/images/../../../etc/passwd`
- **Validation de l'extension** : une application peut exiger que le fichier fourni se termine par une extension attendue. On injecte un octet nul pour forcer la fin du chemin d'accès au fichier avant l'extension :
	- `filename=../../../etc/passwd%00.png`

## Pièges et points d'attention BSCP
- Sur les labs de type « validation que le chemin commence par le dossier attendu », reprendre le répertoire exact vu dans les requêtes normales (par exemple `/var/www/images/`).
- Sur les labs de type « validation de l'extension », prendre une extension déjà utilisée par l'application (`.png`, `.jpg`).
- Le null byte ne fonctionne plus sur les piles modernes : c'est une particularité de lab, à ne tenter qu'après les autres techniques sur une cible réelle.

## Labs PortSwigger
- [[Path-traversal#Lab 5 - Validation du début du chemin]] (Practitioner)
- [[Path-traversal#Lab 6 - Validation de l'extension avec contournement par octet nul]] (Practitioner)

## Liens
- [[Path-traversal]]
- [[Path-traversal-chemin-absolu]]
- [[Path-traversal-sequences-et-encodage]]
