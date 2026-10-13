---
tags: [bscp, server-side, file-upload]
niveau: practitioner
statut: à faire
---
# File upload - liste noire de types de fichiers

## En bref
- Les développeurs peuvent bloquer les types de fichiers dangereux (extensions comme `.php`), mais oublier les divergences d'analyse lors de la vérification des extensions.
- Comme toute liste noire, elle omet forcément des types de fichiers moins courants qui restent dangereux.
- Note parente : [[File-upload]]

## Comment détecter
- Téléverser des fichiers avec différentes extensions et comparer les messages de refus.
- Repérer si le refus porte sur l'extension, sur le `Content-Type` ou sur le contenu.

## Comment exploiter (principe)
- Les techniques de contournement (extensions alternatives, double extension, écrasement de la configuration serveur, etc.) ne sont pas encore étudiées dans ce cours. Cette section sera complétée à ce moment.

## Pièges et points d'attention BSCP
- Une validation peut être robuste à un endroit et absente ailleurs : voir « Application incohérente » dans [[File-upload]].

## Labs PortSwigger
- Labs de ce thème listés comme « à venir » dans [[File-upload]].

## Liens
- [[File-upload]]
- [[File-upload-validation-content-type]]
