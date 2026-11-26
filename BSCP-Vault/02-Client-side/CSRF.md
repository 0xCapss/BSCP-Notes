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
Exemple : une fonction de changement d'adresse e-mail envoie cette requête:
```http
POST /email/change HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 30
Cookie: session=yvthwsztyeQkAPzeQ5gHgTvlyxHfsAfE

email=wiener@normal-user.com
```
Elle remplit les trois conditions :
- Changer l'e-mail intéresse l'attaquant : il peut ensuite déclencher une réinitialisation de mot de passe et prendre le contrôle total du compte.
- Le cookie de session est le seul moyen d'identifier l'utilisateur, il n'y a ni jeton ni autre mécanisme de suivi.
- L'attaquant détermine sans peine les valeurs des paramètres nécessaires.
L'attaquant construit alors une page web avec un formulaire caché qui pointe vers l'action vulnérable. Un script le soumet automatiquement au chargement de la page.
```html
<html>
    <body>
        <form action="https://vulnerable-website.com/email/change" method="POST">
            <input type="hidden" name="email" value="pwned@evil-user.net" />
        </form>
        <script>
            document.forms[0].submit();
        </script>
    </body>
</html>
```
Ainsi quand la victime se rend sur la page de l'attaquant:
1. La page déclenche une requête HTTP vers le site vulnérable.
2. Si la victime est connectée au site, son navigateur ajoute son cookie de session à la requête.
3. Le site traite la requête normalement, la croit émise par la victime et change son adresse e-mail.
Le CSRF est surtout décrit avec une gestion de session par cookies, mais il peut aussi apparaître dès que l'application ajoute automatiquement des identifiants de l'utilisateur aux requêtes, comme avec l'authentification HTTP Basic ou l'authentification par certificat.
### Construire l'exploit avec burp
Écrire le HTML à la main devient pénible quand la requête contient beaucoup de paramètres ou des particularités. Le plus simple est le générateur de CSRF PoC intégré à Burp Suite Professional :
	1. Sélectionner, n'importe où dans Burp, la requête à tester ou à exploiter.
	2. Clic droit, puis Engagement tools / Generate CSRF PoC.
	3. Burp génère le HTML qui déclenche la requête, sans les cookies, que le navigateur de la victime ajoute tout seul.
	4. Ajuster les options du générateur si besoin, notamment pour gérer les requêtes atypiques.
	5. Copier le HTML dans une page web, l'ouvrir dans un navigateur connecté au site vulnérable, et vérifier que la requête part bien et que l'action attendue a lieu.

## Pièges et points d'attention BSCP
- 

## Prévention
- 

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Lab: Vulnérabilité CSRF sans protection
La fonctionnalité de modification de l'adresse e-mail de cet exercice pratique est vulnérable à une attaque CSRF.
Pour résoudre cet exercice, créez un code HTML utilisant une attaque CSRF afin de modifier l'adresse e-mail de l'utilisateur, puis téléchargez-le sur votre serveur d'exploitation.
Vous pouvez vous connecter à votre propre compte à l'aide des identifiants suivants : wiener:peter

## Mes notes
- 

## Liens
- [[XSS]]
- [[CORS]]
- [[Authentication]]
