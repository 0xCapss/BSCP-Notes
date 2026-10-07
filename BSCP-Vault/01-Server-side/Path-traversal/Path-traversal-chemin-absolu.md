---
tags: [bscp, server-side, path-traversal]
niveau: practitioner
statut: à faire
---
# Path traversal - séquences bloquées, contournement par chemin absolu

## En bref
- L'application bloque ou neutralise les séquences `../`, mais utilise le paramètre tel quel comme chemin quand il est absolu.
- Note parente : [[Path-traversal]]

## Comment exploiter (principe)
- Remplacer entièrement la valeur du paramètre par le chemin absolu du fichier visé : `filename=/etc/passwd`.
- Aucune séquence de traversal n'est nécessaire, donc le filtre sur `../` n'a rien à bloquer.

## Pièges et points d'attention BSCP
- À tester en premier quand `../` est refusé : c'est le contournement le plus rapide.
- Ne pas confondre avec la validation de préfixe : voir [[Path-traversal-prefixe-et-extension]] (le chemin doit alors commencer par le répertoire attendu).

## Labs PortSwigger
- [[Path-traversal#Lab 2 - Séquences de traversée bloquées, contournement par chemin absolu]] (Practitioner)

## Liens
- [[Path-traversal]]
- [[Path-traversal-cas-simple]]
- [[Path-traversal-prefixe-et-extension]]
