
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
- Plusieurs méthodes d'implément
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

