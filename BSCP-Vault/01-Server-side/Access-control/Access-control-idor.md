---
tags: [bscp, server-side, access-control]
niveau: apprentice
statut: à faire
---
# Contrôle d'accès - références directes non sécurisées à un objet (IDOR)

## En bref
- Les IDOR surviennent quand l'application utilise directement une entrée fournie par l'utilisateur pour accéder à un objet (fichier, enregistrement) sans vérifier que cet utilisateur en est propriétaire.
- Cas particulier d'escalade horizontale (voir [[Access-control-escalade-horizontale]]) : l'objet est souvent un fichier statique plutôt qu'un identifiant de base de données.
- Note parente : [[Access-control]]

## Comment détecter
- Repérer des noms de fichiers ou numéros incrémentaux dans les URL (`/download-transcript/2.txt`, `/invoices/1042.pdf`).
- Incrémenter ou décrémenter la valeur et comparer les réponses.

## Comment exploiter (principe)
- Exemple : un site stocke les transcriptions de chat sur le serveur avec des noms de fichiers incrémentaux (`2.txt`). Modifier le numéro dans l'URL de téléchargement donne accès aux transcriptions d'autres utilisateurs, qui peuvent contenir des identifiants.
- Le même principe s'applique aux fichiers statiques non protégés (images, exports, factures).

## Pièges et points d'attention BSCP
- Ne pas oublier les fichiers téléchargés : l'IDOR est fréquente sur les exports et les pièces jointes.
- Tester aussi les identifiants bas (`1`), qui appartiennent souvent à des comptes privilégiés.

## Labs PortSwigger
- [[Access-control#Lab 9 - Références directes non sécurisées à un objet]] (Apprentice)

## Liens
- [[Access-control]]
- [[Path-traversal]]
