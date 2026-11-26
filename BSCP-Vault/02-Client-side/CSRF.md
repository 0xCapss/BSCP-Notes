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
### Erreurs courantes dans la validation des jetons CSRF
Les failles CSRF viennent généralement d'une validation défaillante des jetons CSRF. Cette partie du cours passe en revue les erreurs les plus courantes, celles qui permettent à un attaquant de contourner la protection.
**La validation du jeton dépend de la méthode HTTP.** Certaines applications valident correctement le jeton sur une requête POST, mais sautent la validation sur une requête GET. L'attaquant bascule alors en GET pour contourner la protection et mener son attaque :
```http
GET /email/change?email=pwned@evil-user.net HTTP/1.1
Host: vulnerable-website.com
Cookie: session=2yQIDcpia41WrATfjPqvm9tOkDvkMvLm
```

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
### Livrer l'exploit à la victime
Le mécanisme de livraison est en gros le même qu'une reflected XSS. L'attaquant héberge le HTML malveillant sur un site qu'il contôle, puis pousse les victimes à le visiter:
- en leur envoyant le lien par e-mail ou par message sur les réseaux sociaux.
- ou, si l'exploit est posé sur un site très fréquenté (un commentaire d'utilisateur, par exemple), en attendant simplement que les gens passent.
Certaines attaques CSRF simples passent par une requête GET et sont entièrement autonomes : une seule URL du site vulnérable suffit. L'attaquant n'a alors pas besoin d'un site externe, il peut proposer directement une URL malveillante sur le domaine vulnérable. Si le changement d'e-mail de l'exemple accepte GET, l'attaque tient en une ligne :
```html
<img src="https://vulnerable-website.com/email/change?email=pwned@evil-user.net">
```

## Pièges et points d'attention BSCP
- 

## Prévention
Aujourd'hui, détecter et exploiter un CSRF oblige souvent à contourner les défenses du site cible du navigateur de la victime ou les deux. Les trois plus courantes sont:
- **Jeton CSRF.** Valeur unique, secrète et imprévisible, générée côté serveur et partagée avec le client. Pour une action sensible (envoi d'un formulaire, par exemple), le client doit renvoyer le bon jeton. L'attaquant a alors beaucoup de mal à fabriquer une requête valide à la place de la victime.
- **Cookies SameSite.** Mécanisme du navigateur qui décide quand les cookies d'un site sont envoyés dans des requêtes venant d'autres sites. Comme les actions sensibles exigent en général un cookie de session authentifié, une bonne restriction SameSite empêche de les déclencher depuis un autre site. Chrome applique SameSite=Lax par défaut depuis 2021. C'est la norme proposée, donc les autres navigateurs majeurs devraient suivre.
- **Validation du Referer.** Certaines applications vérifient l'en-tête HTTP Referer, en général pour s'assurer que la requête vient de leur propre domaine. C'est en règle générale moins efficace que le jeton CSRF.
#### Jeton CSRF en détail
Un jeton CSRF est une valeur unique, secrète et imprévisible, générée par l'application côté serveur et partagée avec le client. Pour une action sensible comme l'envoi d'un formulaire, le client doit inclure le bon jeton dans sa requête. Sinon, le serveur refuse d'exécuter l'action.
Le moyen courant de transmettre le jeton au client est un paramètre caché dans le formulaire HTML :
```html
<form name="change-email-form" action="/my-account/change-email" method="POST">
    <label>Email</label>
    <input required type="email" name="email" value="example@normal-website.com">
    <input required type="hidden" name="csrf" value="50FaWgdOhi9M9wyna8taR1k3ODOR8d6u">
    <button class='button' type='submit'> Update email </button>
</form>
```
L'envoi de ce formulaire produit la requête suivante :
```http
POST /my-account/change-email HTTP/1.1
Host: normal-website.com
Content-Length: 70
Content-Type: application/x-www-form-urlencoded

csrf=50FaWgdOhi9M9wyna8taR1k3ODOR8d6u&email=example@normal-website.com
```
Bien implémenté, le jeton rend difficile la construction d'une requête valide au nom de la victime : l'attaquant ne peut pas prédire sa valeur, donc ne peut pas l'inclure dans la requête malveillante.
Le jeton n'est pas forcément envoyé en paramètre caché d'une requête POST. Certaines applications le placent dans un en-tête HTTP, par exemple. Le mode de transmission pèse lourd sur la sécurité de l'ensemble du mécanisme.


## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Lab: Vulnérabilité CSRF sans protection
La fonctionnalité de modification de l'adresse e-mail de cet exercice pratique est vulnérable à une attaque CSRF.
Pour résoudre cet exercice, créez un code HTML utilisant une attaque CSRF afin de modifier l'adresse e-mail de l'utilisateur, puis téléchargez-le sur votre serveur d'exploitation.
Vous pouvez vous connecter à votre propre compte à l'aide des identifiants suivants : wiener:peter
#### Solution
1. Open Burp's browser and log in to your account. Submit the "Update email" form, and find the resulting request in your Proxy history.
2. If you're using Burp Suite Professional, right-click on the request and select Engagement tools / Generate CSRF PoC. Enable the option to include an auto-submit script and click "Regenerate".
    Alternatively, if you're using Burp Suite Community Edition, use the following HTML template. You can get the request URL by right-clicking and selecting "Copy URL".
    `<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email"> <input type="hidden" name="email" value="anything@web-security-academy.net"> </form> <script> document.forms[0].submit(); </script>`
3. Go to the exploit server, paste your exploit HTML into the "Body" section, and click "Store".
4. To verify that the exploit works, try it on yourself by clicking "View exploit" and then check the resulting HTTP request and response.
5. Change the email address in your exploit so that it doesn't match your own.
6. Click "Deliver to victim" to solve the lab.

### Lab: CSRF where token validation depends on request method
La fonctionnalité de modification d'adresse e-mail de ce laboratoire est vulnérable aux attaques CSRF. Elle tente de bloquer ces attaques, mais n'applique des mesures de protection qu'à certains types de requêtes.

Pour résoudre cet exercice, utilisez votre serveur d'exploitation pour héberger une page HTML qui utilise une attaque CSRF afin de modifier l'adresse e-mail de l'utilisateur.

Vous pouvez vous connecter à votre propre compte à l'aide des identifiants suivants : wiener:peter
#### Solution


## Mes notes
- 

## Liens
- [[XSS]]
- [[CORS]]
- [[Authentication]]
