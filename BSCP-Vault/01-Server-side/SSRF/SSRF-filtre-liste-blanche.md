---
tags: [bscp, server-side, ssrf]
niveau: expert
statut: à faire
---
# SSRF avec filtre d'entrée basé sur une liste blanche

## En bref
- Certaines applications n'autorisent que les entrées qui correspondent à une liste blanche de valeurs (domaines autorisés).
- Le filtre peut chercher la correspondance au début de l'entrée ou à l'intérieur.
- Note parente : [[SSRF]]

## Pourquoi c'est contournable
- Les fonctionnalités de la spécification des URL sont souvent négligées quand l'analyse et la validation sont faites de façon ad hoc.
- Le code de validation et le code qui effectue la requête HTTP peuvent interpréter la même URL différemment.

## Comment exploiter (principe)
- **Identifiants avant le nom d'hôte** avec `@` : `https://expected-host:fakepassword@evil-host`
- **Fragment d'URL** avec `#` : `https://evil-host#expected-host`
- **Hiérarchie DNS** : placer la valeur attendue dans un nom DNS complet que l'on contrôle : `https://expected-host.evil-host`
- **Encodage d'URL** pour semer la confusion dans l'analyse. Utile surtout si le code du filtre traite les caractères encodés différemment du code qui effectue la requête HTTP.
- **Double encodage** : certains serveurs décodent de manière récursive, ce qui crée d'autres divergences.
- **Combiner plusieurs techniques** : par exemple `http://localhost:80%2523@stock.weliketoshop.net/admin` (le `%2523` devient `#` après deux décodages).

## Pièges et points d'attention BSCP
- Procéder par étapes : tester d'abord `@` seul, puis `#`, puis son double encodage, et noter ce qui est accepté ou refusé à chaque fois.
- Le filtre peut rejeter `#` en clair mais l'accepter encodé : c'est le signe d'un décodage tardif côté requête.

## Labs PortSwigger
- [[SSRF#Lab 6 - SSRF avec filtre d'entrée basé sur une liste blanche]] (Expert)

## Liens
- [[SSRF]]
- [[SSRF-filtre-liste-noire]]
- [[SSRF-redirection-ouverte]]
