---
tags: [bscp, client-side, csrf]
niveau: practitioner
statut: à faire
---
# Cross-site request forgery (CSRF)

## En bref
- L'attaque CSRF est une faille web qui permet à un attaquant d'amener un utilisateur à effectuer des actions qu'il n'a pas l'intention de faire. Elle contourne en partie la same-origin policy, censée empêcher des sites différents d'interférer les uns avec les autres.
- Quand l'attaque réussit, la victime exécute une action à son insu: changer son adresse mail, changer son mot de passe, faire un virement bancaire... Selon l'action, l'attaquant peut prendre le contrôle total du compte. Si la victime a un rôle privilégié, il peut prendre le contrôle de toutes les données et fonctionnalités de l'application.

## Types et variantes
- 

## Comment détecter
- 

## Comment exploiter (principe)
- 3 conditions doivent être réunis pour qu'une CSRF soit possible:
1. **Une action pertinente.** L'application contient une action que l'attaquant a intérêt à provoquer. Soit une action privilégiée (modifier les permissions d'autres utilisateurs), soit une action sur les données propres à l'utilisateur (changer son mot de passe).
2. **Une gestion de session basée sur les cookies.** L'action passe par une ou plusieurs requêtes HTTP, et l'application s'appuie uniquement sur les cookies de session pour identifier l'utilisateur. Aucun autre mécanisme ne suit la session ou ne valide les requêtes.
3. **Aucun paramètre imprévisible.** Les requêtes qui déclenchent l'action ne contiennent aucun paramètre dont l'attaquant ne peut pas déterminer ou deviner la valeur. Exemple : une fonction de changement de mot de passe n'est pas vulnérable si l'attaquant doit connaître le mot de passe actuel.

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
- [[XSS]]
- [[CORS]]
- [[Authentication]]
