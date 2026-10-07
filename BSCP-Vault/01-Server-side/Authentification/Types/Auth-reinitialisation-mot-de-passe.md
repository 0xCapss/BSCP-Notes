---
tags: [bscp, server-side, authentication]
niveau: practitioner
statut: à faire
---
# Authentification - réinitialisation du mot de passe

## En bref
- Fonctionnalité risquée par nature : elle doit authentifier l'utilisateur par un autre moyen que le mot de passe.
- Note parente : [[Autres-mecanismes-authentification]]

## Comment détecter
- Observer le lien reçu par e-mail : un paramètre identifie-t-il l'utilisateur (`user=`) ? Le token est-il long et imprévisible ?
- Tester si le token est revalidé à la soumission finale.
- Tester `X-Forwarded-Host` et `Host` sur `POST /forgot-password` (voir [[HTTP-Host-header]]).

## Comment exploiter (principe)
- Fonctionnalité risquée par nature : elle doit authentifier l'utilisateur par un autre moyen que le mot de passe, ce qui ajoute une surface d'attaque.
- Elle doit impérativement être implémentée de façon sécurisée, sous peine de permettre à un attaquant de prendre le contrôle d'un compte sans connaître le mot de passe initial.
- Plusieurs méthodes d'implémentation existent, avec des niveaux de risque différents selon la conception retenue.
- Un site qui gère correctement ses mots de passe ne devrait jamais être capable d'envoyer le mot de passe actuel par e-mail : cela signifierait qu'il le stocke en clair ou de façon réversible.
- Certains sites contournent ce problème en générant un nouveau mot de passe temporaire envoyé par e-mail.
	- Envoyer un mot de passe permanent par un canal non chiffré est à éviter : s'il n'expire pas rapidement ou si l'utilisateur ne le change pas immédiatement, l'approche devient vulnérable à une interception (man-in-the-middle).
	- L'e-mail n'est pas un canal sécurisé : les boîtes de réception sont permanentes, mal adaptées au stockage d'informations confidentielles, et très souvent synchronisées sur plusieurs appareils.
- Envoyer une URL unique vers une page de réinitialisation est une méthode plus sûre que l'envoi direct d'un mot de passe.
- Exemple d'implémentation faible, car basée sur un paramètre prévisible :
	`http://vulnerable-website.com/reset-password?user=victim-user`
	- Si ce paramètre est modifiable, un attaquant peut le remplacer par n'importe quel nom d'utilisateur identifié et accéder à la page de réinitialisation du compte visé, sans jamais avoir reçu le lien.
- Une meilleure implémentation utilise un token à forte entropie pour construire l'URL de réinitialisation.
	- L'URL ne doit communiquer aucun indice sur l'identité de l'utilisateur ciblé.
	- Le serveur doit vérifier l'existence du token côté back-end pour retrouver l'utilisateur associé, le faire expirer rapidement et le détruire une fois le mot de passe changé.
- Certains sites ne revalident pas le token au moment de la soumission du formulaire. Un attaquant peut alors accéder au formulaire avec son propre token, le supprimer de la requête, et réinitialiser le mot de passe d'un autre utilisateur.
- Si l'URL de l'e-mail de réinitialisation est générée dynamiquement (par exemple à partir d'un en-tête `Host` ou `X-Forwarded-Host`), elle peut être vulnérable au **password reset poisoning** : un attaquant piège le lien envoyé à la victime pour qu'il pointe vers un domaine qu'il contrôle, et récupère ainsi le token de la victime.

## Pièges et points d'attention BSCP
- Toujours tester `X-Forwarded-Host` (et `X-Forwarded-Server`, `X-Host`) sur les endpoints qui génèrent un lien envoyé par e-mail.
- Ne pas confondre le token volé (visible dans les logs de l'exploit server) et le token légitime reçu dans son propre e-mail : il faut combiner les deux.
- Retirer le token de l'URL **et** du corps de la requête quand on teste sa revalidation.

## Labs PortSwigger
- [[Autres-mecanismes-authentification#Lab 1 - Empoisonnement de la réinitialisation du mot de passe via un middleware]] (Practitioner)
- [[Authentication#Lab 9 - Logique défaillante de réinitialisation du mot de passe]] (Apprentice)

## Liens
- [[Autres-mecanismes-authentification]]
- [[HTTP-Host-header]]
- [[Auth-modification-mot-de-passe]]
