---
tags: [bscp, server-side, file-upload]
niveau: practitioner
statut: à faire
---
# File upload vulnerabilities

## En bref
- Elles surviennent quand un serveur web permet de téléverser des fichiers dans son système de fichiers sans valider suffisamment leur nom, leur type et encore moins leur contenu ni leur taille.
- Sans restrictions, même une simple fonction de téléversement d'images peut servir à envoyer des fichiers arbitraire et potentiellement dangereux.
- Cela peut inclure des fichiers de script côté serveur, qui permettent l'exécution de code à distance.

## Types et variantes
- 

## Comment détecter
- 

## Comment exploiter (principe)
### Origine des vulnérabilités de téléchargements de fichiers
- Les sites sans aucune restriction sont rares, vu les dangers évidents.
- Le plus souvent, les développeurs mettent en place une validation supposés robuste, mais souvent elle est soit défaillante, soit facilement contournable.
### Liste noire de type de fichiers
- Les développeurs peuvent bloquer les types de fichiers dangereux, mais oublier les divergences d''analyse lors de la vérification des extensions.
- Comme toute liste liste, elle omet forcément des types de fichiers moins courants qui restent dangereux.
### Vérification des propriétés manipulables
- Le site peut vérifier le type de fichier à partir de propriétés et l'attaquant peut facilement modifier avec l'outil Repeater.
### Application incohérente
- Même une validation robuste peut être appliquée de manière incohérente à travers le réseau d'hébergeurs et de répertoires qui composent le site.
## Exploiter les téléchargements sans restriction pour déployer un web shell
- Pire scénario: le site permet de téléverser des scripts côté serveur (PHP, Java, Python) et est configuré pour les exécuter comme du code.
- Il devient alors super facile de créer son propre web shell sur le serveur
- **Web shell**
	- Script malveillant qui permet d’exécuter des commandes arbitraires sur un serveur web distant, simplement en envoyant des requêtes HTTP vers le bon point de terminaison.
	- Cela donne de fait un contrôle total sur le serveur
	- Il permet de lire ou écrire des fichiers arbitraires.
	- On peut exfiltrer des données sensibles.
	- Utilise le serveur comme pivot pour attaquer l'infrastructure interne et d'autres serveurs extérieurs au réseau.
- Exemple en PHP:
	- Lecture de fichier : `<?php echo file_get_contents('/path/to/target/file'); ?>`. Une requête vers le fichier téléversé renvoie le contenu du fichier ciblé dans la réponse.
	- 

## Pièges et points d'attention BSCP
- 

## Prévention
- 

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
- 

## Mes notes
- 

## Liens
- [[Path-traversal]]
- [[Command-injection]]
- [[Access-control]]
