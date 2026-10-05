
## En bref
- Au-delà de la page de connexion elle-même, les fonctionnalités annexes liées à l'authentification (rester connecté, réinitialiser le mot de passe, changer le mot de passe) manipulent les mêmes informations sensibles mais sont souvent moins auditées.
- Elles constituent donc une surface d'attaque à part entière : un défaut sur l'une d'elles permet de contourner l'authentification principale sans jamais avoir à en casser la logique.

## Types et variantes
- La plupart des sites web proposent des fonctionnalités supplémentaires pour gérer le compte, comme :
	- La modification du mot de passe.
	- La réinitialisation du mot de passe en cas d'oubli.
- Ces mécanismes sont une source fréquente de vulnérabilités, car on oublie facilement qu'ils doivent être rendus aussi robustes que la page de connexion principale.

### Maintenir la connexion des utilisateurs
- Fonctionnalité qui permet à un utilisateur de rester connecté après avoir fermé son navigateur, via un token stocké dans un cookie persistant.
- Ce cookie peut être généré par le site à partir de valeurs statiques (par exemple le nom d'utilisateur suivi d'un horodatage) : un attaquant qui observe plusieurs cookies peut en déduire la structure de génération.
- Le cookie peut être chiffré, mais un encodage réversible comme le base64 n'offre aucune protection réelle.
- Si le mot de passe est haché sans salt, des listes de mots de passe courants (rainbow tables) permettent de le retrouver. D'où l'importance du salt.
- Via une faille XSS, un attaquant peut dérober le cookie "Se souvenir de moi" d'un autre utilisateur et en étudier la structure pour le forger.

### Réinitialisation du mot de passe
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

### Modification du mot de passe utilisateur
- Le processus classique demande le mot de passe actuel, puis le nouveau mot de passe saisi deux fois.
- Ces pages reposent sur le même mécanisme de vérification qu'une page de connexion classique, et sont donc exposées aux mêmes techniques d'attaque (énumération, brute-force, etc.).
- Cas typique de faille : le nom d'utilisateur est transmis dans un champ masqué du formulaire. Un attaquant peut modifier cette valeur dans la requête pour cibler un utilisateur arbitraire, sans être connecté à sa victime.

## Comment détecter
- Récupérer plusieurs cookies "se souvenir de moi" (y compris via XSS si possible) et chercher un motif prévisible (encodage réversible, concaténation username + timestamp, absence de signature).
- Sur la fonctionnalité de réinitialisation : vérifier si un paramètre (`user`, `username`, `email`) identifie la cible dans l'URL ou le corps de la requête, et s'il est modifiable.
- Avec Burp, intercepter la requête `POST /forgot-password` et tester la prise en compte d'en-têtes comme `Host` ou `X-Forwarded-Host` dans la génération du lien envoyé par e-mail.
- Vérifier si le token de réinitialisation est revalidé au moment de la soumission finale du nouveau mot de passe, ou seulement à l'affichage du formulaire.
- Sur la fonctionnalité de changement de mot de passe : vérifier la présence du nom d'utilisateur en champ masqué, et comparer les messages d'erreur retournés selon que le mot de passe actuel ou les nouveaux mots de passe sont corrects ou non (une différence de message permet l'énumération).

## Comment exploiter (principe)
- **Password reset poisoning via en-tête d'hôte** : ajouter ou modifier l'en-tête `X-Forwarded-Host` (ou `Host`) de la requête `POST /forgot-password` avec un domaine contrôlé par l'attaquant (par exemple un exploit server), puis soumettre le `username` de la victime. Le lien envoyé par e-mail à la victime pointe alors vers ce domaine ; la requête que la victime déclenche en cliquant (ou que l'attaquant intercepte dans les logs d'accès du domaine contrôlé) contient le token valide de la victime, réutilisable pour réinitialiser son mot de passe.
- **Contournement par paramètre prévisible** : remplacer directement le paramètre identifiant l'utilisateur (`user=victim`) dans la requête de réinitialisation ou de changement de mot de passe par le nom d'utilisateur de la cible.
- **Brute-force via message d'erreur distinctif** : sur le changement de mot de passe, soumettre un mot de passe actuel correct et deux nouveaux mots de passe différents entre eux ; si le message d'erreur retourné ("Current password is incorrect" vs "New passwords do not match") diffère selon que le mot de passe actuel testé est bon ou mauvais, le point devient une oracle d'énumération exploitable avec Burp Intruder en forçant le paramètre `current-password` tout en fixant l'username à celui de la victime.
- **Vol de token via XSS** : si une faille XSS existe par ailleurs, l'utiliser pour exfiltrer le cookie "se souvenir de moi" d'un utilisateur ciblé et en déduire ou réutiliser sa structure.

## Pièges et points d'attention BSCP
- Toujours tester la prise en compte de `X-Forwarded-Host` (et variantes : `X-Forwarded-Server`, `X-Host`) sur les endpoints qui génèrent un lien envoyé par e-mail : c'est le vecteur le plus courant de password reset poisoning sur ces labs.
- Ne pas confondre le token "volé" côté attaquant (visible dans les logs de son propre exploit server) et le token légitime reçu par la victime dans son e-mail : il faut combiner les deux pour reconstituer une réinitialisation valide.
- Vérifier systématiquement si un champ masqué (hidden input) transporte le nom d'utilisateur sur les formulaires de changement de mot de passe : c'est un contournement d'authentification simple à manquer.
- Lors d'un brute-force avec Intruder, bien distinguer les deux nouveaux mots de passe (`new-password-1` et `new-password-2`) pour qu'ils restent différents entre eux : c'est cette différence qui permet d'obtenir un message d'erreur exploitable plutôt qu'un verrouillage de compte.
- Ne pas oublier de revérifier la revalidation du token de réinitialisation à la soumission finale, pas uniquement à l'ouverture du formulaire.

## Prévention
- Plusieurs principes permettent de réduire le risque sur ces fonctionnalités annexes :
	- Protéger les identifiants des utilisateurs :
		- Ne jamais transmettre de données de connexion sur une connexion non chiffrée (HTTPS).
		- Vérifier qu'aucun nom d'utilisateur ni adresse e-mail n'est exposé, que ce soit via des profils publics ou des réponses HTTP qui les reflètent.
	- Ne pas compter sur les utilisateurs pour assurer la sécurité :
		- Une authentification stricte demande un effort aux utilisateurs, qui chercheront à l'éviter.
		- Imposer les comportements sécurisés plutôt que les suggérer.
	- Politique de mot de passe :
		- Les politiques traditionnelles échouent souvent : les utilisateurs adaptent des mots de passe prévisibles pour satisfaire les règles imposées.
		- Une alternative plus efficace : un vérificateur de mot de passe qui évalue la solidité en temps réel pendant la saisie.
		- N'autoriser que les mots de passe jugés sûrs par ce vérificateur impose des mots de passe robustes plus efficacement que des règles classiques.
	- Empêcher l'énumération des noms d'utilisateurs :
		- Imposer des messages d'erreur génériques, identiques dans tous les scénarios.
		- Renvoyer systématiquement le même code d'état HTTP.
		- Veiller à ce que les temps de réponse soient aussi difficiles à distinguer que possible selon les scénarios.
	- Protection contre les attaques par force brute :
		- Limiter strictement le nombre de tentatives par utilisateur, sans se baser uniquement sur l'adresse IP.
		- Empêcher les attaquants de manipuler leur adresse IP apparente.
		- Idéalement, exiger un CAPTCHA une fois une limite atteinte.
		- L'objectif : rendre le processus aussi fastidieux que possible pour que l'attaquant abandonne et se tourne vers une cible plus facile.
	- Vérifier trois fois sa logique de validation :
		- De simples failles logiques se glissent facilement dans le code. Auditer minutieusement toute logique de vérification ou de validation pour les éliminer.
	- Ne pas oublier les fonctionnalités complémentaires :
		- Ne pas se concentrer uniquement sur la page de connexion principale : les fonctionnalités annexes liées à l'authentification sont tout aussi exposées.
		- La réinitialisation et la modification du mot de passe constituent une surface d'attaque aussi valable que la connexion principale, et doivent être tout aussi robustes.
	- Mettre en place une authentification multifactorielle adéquate :
		- 2FA par SMS : techniquement deux facteurs (ce que l'on connaît et ce que l'on possède), mais vulnérable au SIM swapping et au phishing en temps réel.
		- Préférer une application dédiée qui génère directement le code de vérification.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-05
- [x] Practitioner ✅ 2026-10-05

## Journal des labs

### Lab : Password reset poisoning via middleware
Ce lab est vulnérable au password reset poisoning. L'utilisateur `carlos` clique sans vigilance sur tout lien reçu par e-mail. Objectif : se connecter au compte de Carlos. Compte perso : `wiener:peter`. Les e-mails envoyés à ce compte sont lisibles via le client mail de l'exploit server.

1. Avec Burp actif, observer la fonctionnalité de mot de passe oublié : un lien contenant un token de réinitialisation unique est envoyé par e-mail.
2. Envoyer la requête `POST /forgot-password` à Burp Repeater. Constater que l'en-tête `X-Forwarded-Host` est pris en compte et permet de rediriger le lien de réinitialisation généré dynamiquement vers un domaine arbitraire.
3. Récupérer l'URL de son propre exploit server.
4. Dans Repeater, ajouter l'en-tête suivant à la requête :
	`X-Forwarded-Host: VOTRE-ID-EXPLOIT-SERVER.exploit-server.net`
5. Remplacer le paramètre `username` par `carlos` et envoyer la requête.
6. Sur l'exploit server, consulter le journal d'accès (access log) : une requête `GET /forgot-password` y apparaît, contenant le token de la victime en paramètre. Le noter.
7. Dans son propre client mail, copier le lien de réinitialisation légitime (pas celui qui pointe vers l'exploit server), le coller dans le navigateur, puis remplacer la valeur du paramètre `temp-forgot-password-token` par le token volé à la victime.
8. Charger cette URL et définir un nouveau mot de passe pour le compte de Carlos.
9. Se connecter au compte de Carlos avec ce nouveau mot de passe pour valider le lab.

### Lab : Password brute-force via password change
La fonctionnalité de changement de mot de passe de ce lab est vulnérable au brute-force. Objectif : utiliser une liste de mots de passe candidats pour retrouver celui de Carlos et accéder à sa page "My account".

- Compte perso : `wiener:peter`
- Compte victime : `carlos`
- [Liste des mots de passe candidats](https://portswigger.net/web-security/authentication/auth-lab-passwords)

1. Avec Burp actif, se connecter et tester la fonctionnalité de changement de mot de passe. Constater que le nom d'utilisateur est transmis via un champ masqué (hidden input) de la requête.
2. Observer le comportement en cas de mot de passe actuel erroné : si les deux nouveaux mots de passe saisis correspondent, le compte est verrouillé. Mais si les deux nouveaux mots de passe diffèrent, le message retourné est simplement `Current password is incorrect`. Si le mot de passe actuel est correct mais les deux nouveaux diffèrent, le message devient `New passwords do not match`. Cette différence de message permet d'énumérer le bon mot de passe actuel.
3. Saisir son propre mot de passe actuel correct et deux nouveaux mots de passe différents entre eux. Envoyer cette requête `POST /my-account/change-password` à Burp Intruder.
4. Dans Intruder, remplacer le paramètre `username` par `carlos` et placer un payload sur le paramètre `current-password`, en laissant les deux nouveaux mots de passe différents. Exemple :
	`username=carlos&current-password=§mot-de-passe-incorrect§&new-password-1=123&new-password-2=abc`
5. Dans le panneau **Payloads**, charger la liste des mots de passe candidats comme jeu de payloads.
6. Dans **Settings**, ajouter une règle de grep match pour repérer les réponses contenant `New passwords do not match`, puis lancer l'attaque.
7. À la fin de l'attaque, une seule réponse doit correspondre à ce message. Noter le mot de passe candidat associé.
8. Dans le navigateur, se déconnecter de son propre compte et se reconnecter avec le nom d'utilisateur `carlos` et le mot de passe identifié.
9. Cliquer sur **My account** pour valider le lab.

## Mes notes
- 

## Liens
- [[Authentication]]
- [[Access-control]]
- [[Multi-factor-authentication]]
