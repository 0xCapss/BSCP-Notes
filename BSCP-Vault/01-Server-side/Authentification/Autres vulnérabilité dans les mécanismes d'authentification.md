
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
## Modififcation du mot de passe utilisateur
- Le processus actuel demande généralement le mot de passe actuel puis le nouveau mot de passe 2x.
- Ces pages reposent sur le même mécanisme de vérification qu'une page de connexion classique.
- Elles sont donc exposées aux même techniques d'attaque que les pages de connexion.

- Ainsi, cette fonctionnalité devient dangereuse car un attaquant peut y accéder directement, sans être connecté à sa victime.
- Cas typique : le nom d'utilisateur est transmis dans un champ masqué du formulaire. L'attaquant peut modifier cette valeur dans la requête pour cibler des utilisateurs arbitraires.
## Comment détecter


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


Soluce Lab:
### Lab: Password reset poisoning via middleware

This lab is vulnerable to password reset poisoning. The user `carlos` will carelessly click on any links in emails that he receives. To solve the lab, log in to Carlos's account. You can log in to your own account using the following credentials: `wiener:peter`. Any emails sent to this account can be read via the email client on the exploit server.

1. With Burp running, investigate the password reset functionality. Observe that a link containing a unique reset token is sent via email.
2. Send the `POST /forgot-password` request to Burp Repeater. Notice that the `X-Forwarded-Host` header is supported and you can use it to point the dynamically generated reset link to an arbitrary domain.
3. Go to the exploit server and make a note of your exploit server URL.
4. Go back to the request in Burp Repeater and add the `X-Forwarded-Host` header with your exploit server URL:
    
    `X-Forwarded-Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net`
5. Change the `username` parameter to `carlos` and send the request.
6. Go to the exploit server and open the access log. You should see a `GET /forgot-password` request, which contains the victim's token as a query parameter. Make a note of this token.
7. Go back to your email client and copy the valid password reset link (not the one that points to the exploit server). Paste this into the browser and change the value of the `temp-forgot-password-token` parameter to the value that you stole from the victim.
8. Load this URL and set a new password for Carlos's account.
9. Log in to Carlos's account using the new password to solve the lab.

### Lab: Password brute-force via password change

This lab's password change functionality makes it vulnerable to brute-force attacks. To solve the lab, use the list of candidate passwords to brute-force Carlos's account and access his "My account" page.

- Your credentials: `wiener:peter`
- Victim's username: `carlos`
- [Candidate passwords](https://portswigger.net/web-security/authentication/auth-lab-passwords)

1. With Burp running, log in and experiment with the password change functionality. Observe that the username is submitted as hidden input in the request.
2. Notice the behavior when you enter the wrong current password. If the two entries for the new password match, the account is locked. However, if you enter two different new passwords, an error message simply states `Current password is incorrect`. If you enter a valid current password, but two different new passwords, the message says `New passwords do not match`. We can use this message to enumerate correct passwords.
3. Enter your correct current password and two new passwords that do not match. Send this `POST /my-account/change-password` request to Burp Intruder.
4. In Burp Intruder, change the `username` parameter to `carlos` and add a payload position to the `current-password` parameter. Make sure that the new password parameters are set to two different values. For example:
    
    `username=carlos&current-password=§incorrect-password§&new-password-1=123&new-password-2=abc`
5. In the **Payloads** side panel, enter the list of passwords as the payload set.
6. Click  **Settings** to open the **Settings** side panel, then add a grep match rule to flag responses containing `New passwords do not match`. Start the attack.
7. When the attack finished, notice that one response was found that contains the `New passwords do not match` message. Make a note of this password.
8. In the browser, log out of your own account and lock back in with the username `carlos` and the password that you just identified.
9. Click **My account** to solve the lab.

## Journal des labs
- 

## Mes notes
- 

