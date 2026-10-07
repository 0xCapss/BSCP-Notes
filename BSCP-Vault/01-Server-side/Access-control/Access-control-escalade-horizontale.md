---
tags: [bscp, server-side, access-control]
niveau: apprentice
statut: à faire
---
# Contrôle d'accès - escalade horizontale de privilèges

## En bref
- Un utilisateur accède aux ressources d'un autre utilisateur de même niveau de privilège.
- Très proche de l'IDOR (voir [[Access-control-idor]]).
- Une escalade horizontale peut devenir verticale si la ressource atteinte appartient à un compte administrateur.
- Note parente : [[Access-control]]

## Comment détecter
- Repérer un identifiant de ressource dans l'URL ou le corps (`id`, `userId`, `account`) sur les pages de profil, factures, messages.
- Se connecter avec un compte, puis remplacer l'identifiant par celui d'un autre utilisateur.
- Quand les identifiants sont des GUID, chercher où ils fuient (articles de blog signés, commentaires, messages, exports).

## Comment exploiter (principe)
- C'est le cas où un utilisateur peut accéder à des ressources d'un autre utilisateur. On peut utiliser les mêmes méthodes d'exploitation que pour l'escalade verticale. Par exemple, un utilisateur peut accéder à la page de son compte à l'aide de l'URL suivante :
`https://insecure-website.com/myaccount?id=123`

L'utilisateur peut ainsi modifier son identifiant pour afficher un autre utilisateur et ainsi gagner l'accès à un autre compte. Cette technique est très similaire à celle de l'IDOR. Dans certaines applications, le paramètre exploitable n'a pas de valeur prévisible : au lieu d'un numéro croissant, une application peut utiliser des identifiants uniques globaux (GUID) pour identifier les utilisateurs, ce qui peut empêcher un attaquant de deviner ou de prédire l'identifiant d'un autre utilisateur.
- **Horizontale vers verticale** : si l'identifiant visé est celui d'un administrateur, la page de compte peut révéler son mot de passe ou une clé d'API, ce qui donne un accès privilégié.
- **Fuite dans une redirection** : le serveur peut répondre par une redirection vers la page de connexion tout en incluant les données sensibles dans le corps de la réponse de redirection. Lire le corps de la réponse `302` dans Burp.

## Pièges et points d'attention BSCP
- Un identifiant non prévisible (GUID) ne signifie pas que le contrôle d'accès est correct : si l'ID est exposé ailleurs, l'escalade reste exploitable.
- Ne pas s'arrêter à la première preuve d'accès : vérifier l'étendue réelle (compte administrateur ?).
- Un mot de passe peut être masqué à l'écran mais présent en clair dans le HTML : inspecter la source.

## Labs PortSwigger
- [[Access-control#Lab 5 - Identifiant d'utilisateur contrôlé par un paramètre de requête]] (Apprentice)
- [[Access-control#Lab 6 - Identifiant d'utilisateur contrôlé par un paramètre, avec identifiants imprévisibles]] (Apprentice)
- [[Access-control#Lab 7 - Identifiant d'utilisateur contrôlé par un paramètre, avec fuite de données dans une redirection]] (Apprentice)
- [[Access-control#Lab 8 - Identifiant d'utilisateur contrôlé par un paramètre, avec divulgation du mot de passe]] (Apprentice)

## Liens
- [[Access-control]]
- [[Access-control-idor]]
- [[Authentication]]
