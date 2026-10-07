---
tags: [bscp, server-side, ssrf, oast]
niveau: practitioner
statut: à faire
---
# SSRF aveugle (blind)

## En bref
- Elle survient quand on peut forcer l'application à envoyer une requête HTTP vers une URL fournie, mais que la réponse n'apparaît pas dans ce que renvoie l'interface.
- Plus difficile à exploiter qu'une SSRF classique.
- Leur impact est souvent moindre que celui des SSRF « pleinement informées », à cause de leur nature unidirectionnelle : elles ne permettent pas d'extraire facilement des données sensibles des systèmes back-end.
- Elle peut parfois conduire à l'exécution de code à distance sur le serveur ou sur d'autres composants du back-end.
- Note parente : [[SSRF]]

## Comment détecter
- La méthode la plus fiable repose sur les techniques hors bande (OAST).
- Principe : tenter de déclencher une requête HTTP vers un système externe que l'on contrôle, puis surveiller les interactions réseau avec ce système.
- **Burp Collaborator**
	- C'est l'outil le plus simple et le plus efficace pour l'OAST.
	- Il génère des noms de domaine uniques, à envoyer comme charges utiles à l'application, puis surveille toute interaction avec ces domaines.
	- Une requête HTTP entrante provenant de l'application indique qu'elle est vulnérable à la SSRF.
- **Requête DNS sans requête HTTP**
	- Il est fréquent d'observer une requête DNS pour le domaine Collaborator sans requête HTTP ensuite.
	- Cause habituelle : l'application a tenté la requête HTTP, ce qui a déclenché la requête DNS, mais un filtrage réseau a bloqué la requête HTTP elle-même.
	- L'infrastructure autorise couramment le trafic DNS sortant, nécessaire à de nombreux usages, mais bloque les connexions HTTP vers des destinations inattendues.

## Comment exploiter (principe)
- Détecter une SSRF aveugle capable de déclencher des requêtes hors bande ne suffit pas à garantir qu'elle est exploitable.
- La réponse de la requête back-end étant invisible, ce comportement ne permet pas d'explorer le contenu des systèmes accessibles au serveur d'applications.
- **Méthode 1 : rechercher d'autres vulnérabilités**
	- La SSRF peut servir à chercher des vulnérabilités sur le serveur lui-même ou sur d'autres systèmes back-end.
	- Il est possible de balayer à l'aveugle l'espace d'adresses IP interne avec des charges utiles conçues pour détecter des vulnérabilités bien connues (exemple : Shellshock).
	- Si ces charges utiles emploient aussi des techniques hors bande, on peut découvrir une vulnérabilité critique sur un serveur interne non patché.
- **Méthode 2 : réponses malveillantes**
	- Amener l'application à se connecter à un système contrôlé par l'attaquant, qui renvoie des réponses malveillantes au client HTTP à l'origine de la connexion.
	- Si une grave vulnérabilité côté client existe dans l'implémentation HTTP du serveur, elle peut permettre une exécution de code à distance au sein de l'infrastructure de l'application.

## Pièges et points d'attention BSCP
- Attendre quelques secondes avant de relever Collaborator : le traitement côté serveur est souvent asynchrone.
- Les labs PortSwigger bloquent les interactions avec les systèmes externes arbitraires : utiliser uniquement le serveur public par défaut de Burp Collaborator.
- Surfaces fréquentes de SSRF aveugle : voir [[SSRF-surfaces-cachees]] (en-tête `Referer` notamment).

## Labs PortSwigger
- [[SSRF#Lab 5 - SSRF aveugle avec détection hors bande]] (Practitioner)
- [[SSRF#Lab 7 - SSRF aveugle avec exploitation de Shellshock]] (Expert)

## Liens
- [[SSRF]]
- [[SSRF-surfaces-cachees]]
- [[SSRF-systemes-back-end]]
