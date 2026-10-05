
## En bref
- 

## Types et variantes
- La plupart des sites web propose des fonctionnalités supplémentaires pour gérer leur compte comme:
	- La modification de leur mot de passe
	- Le réinitialiser quand ils l'ont oublié.
- Ces différents mécanismes sont des grandes sources de vulnérabilités car on oublie facilement qu'il faut prendre des mesures sur ces fonctionnalités afin qu'elles soient robustes.
### Maintenir la connexion des utilisateurs
- Fonctionnalité qui consiste à permettre aux utilisateurs de rester connectés après avoir fermé leur session dans le navigateur.
- Token qui est stocké dans un cookie persistant.
- Ce cookie peut être gérer par le site web lui-même et peut être générer par des valeurs statique comme le nom d'utilisateur suivi d'un horodatage. Ainsi, un attaquant peut analyser son cookie et en déduire comment ils sont générés.
- Le cookie peut être également chiffré mais le fait d'utilisé un code bidirectionnel comme la base64 n'offre aucune protection.
- Il se peut que le mot de passe soit haché. Mais il existe une liste de mot de passes bien connu en ligne qui permettent de casser ces mots de passes. Cela montre l'importance du "salt" dans un mot de passe.
- A l'aide de technique comme la XSS, un attaquant peut dérober le cookie "Se souvenir de moi" d'un autre user et en déduire la structure.
### Reset le mot de passe utilisateur
- La réinitialisation d'un mot de passe est une fonctionnalité risqué. Elle doit authentifier un utilisateur par un autre moyen que le mot de passe, ce qui crée une surface d'attaque en plus.
- Elle se doit impérativement d'être implémentée de façon sécurisée, sous peine de permettre un attaquant de prendre le contrôle d'un compte sans nécessairement connaitre le mot de passe initial. 
- Plusieurs méthodes d'implémentation existent, avec des niveaux de vulnérabilité différents selon la conception retenue.
- Si un site web gère correctement ses mots de passe, il ne devrait jamais être capable d'envoyer le mot de passe actuel par e-mail (cela signifie qu'il le stocke en clair ou de façon réversible).
- Certains sites contournent cela en générant un nouveau mot de passe temporaire envoyé par e-mail à la place.
- Envoyer un mot de passe permanent via un canal non sécurisé est à éviter car si ce mot de passe n'expire pas vite ou si le'utilisateur le ne change pas immédiatement, cette approche devient vulnérable aux attaques man-in-the-middle.
- L'e-mail n'est pas considéré comme un canal sécurisé: les boîtes de réception sont permanentes, très mal adaptées au stockage d'information confidentielles, et sont très souvent synchroniser sur d'autres appareils.
- L'envoi d'une URL unique vers une page de réinitialisation est une méthode plus sûre que l'envoi de mot de passe.
- Exemple d'une implémentation faible car elle utilise un paramètre prévisible:
`http://vulnerable-website.com/reset-password?user=victim-user`
- Si ce paramètre est modifiable, un attaquant peut le remplacer par n'importe quel nom d'utilisateur identifié et accéder légitimement à la page de réinitialisation du compte sans jamais avoir reçu le lien.
- Une meilleur implémentation est d'utiliser un token à forte entropie pour construire un URL de réinitialisation.
- L'URL ne doit communiquer aucun indice sur l'identité de l'utilisateur ciblé par la réinitialisation.
- Le serveur doit vérifier l'existence de ce token en back-end pour retrouver l'utilisateur associé, le faire expirer rapidement et le détruire une fois le mot de passe changé.
- Certains sites ne revalident pas le jeton au moment de la soumission du formulaire. Ainsi, un attaquant peut alors accéder au formulaire avec son propre jeton, le supprimer de sa requête et alors réinitialiser un mot de passe utilisateur.
- Si l'URL figurant dans l'e-mail de réinitialisation est générée de manière dynamique, elle peut également être vulnérable à une attaque de type « password reset poisoning ». Dans ce cas, un pirate pourrait potentiellement voler le jeton d'un autre utilisateur et l'utiliser pour modifier son mot de passe.
## Comment détecter
- 

## Comment exploiter (principe)
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

