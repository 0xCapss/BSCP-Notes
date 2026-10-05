
## En bref
- Les fonctionnalités annexes d'authentification (cookie "Se souvenir de moi", réinitialisation et modification du mot de passe) forment une surface d'attaque aussi importante que la page de connexion, mais elles sont souvent moins bien protégées.
- Une faille dans l'une d'elles permet de prendre le contrôle d'un compte sans connaître son mot de passe, ou de contourner les protections de la page de connexion (limitation de tentatives, verrouillage de compte).

## Types et variantes
- La plupart des sites proposent des fonctionnalités supplémentaires pour gérer un compte :
	- Modifier son mot de passe.
	- Le réinitialiser en cas d'oubli.
	- Rester connecté d'une session à l'autre.
- Ces mécanismes sont une grande source de vulnérabilités, car on oublie facilement de les durcir autant que la connexion principale.

### Maintenir la connexion des utilisateurs
- Fonctionnalité qui permet de rester connecté après la fermeture du navigateur, généralement grâce à un jeton stocké dans un cookie persistant.
- Si le cookie est généré à partir de valeurs prévisibles (nom d'utilisateur suivi d'un horodatage, par exemple), un attaquant peut analyser son propre cookie, en déduire la construction, puis forger celui d'une autre personne.
- Un encodage réversible comme le base64 n'est pas du chiffrement et n'offre aucune protection.
- Si le mot de passe est simplement haché dans le cookie, il peut être retrouvé grâce aux tables de hachages de mots de passe courants disponibles en ligne. Cela montre l'importance du sel (salt) : un hachage sans sel est beaucoup plus facile à casser.
- Une faille comme la XSS permet de voler le cookie "Se souvenir de moi" d'un autre utilisateur, et donc d'en déduire la structure.

### Réinitialiser le mot de passe
- La réinitialisation est une fonctionnalité risquée : elle doit authentifier l'utilisateur autrement que par son mot de passe, ce qui crée une surface d'attaque supplémentaire.
- Plusieurs méthodes d'implémentation existent, avec des niveaux de vulnérabilité différents :
	- **Envoi du mot de passe actuel par e-mail** : ne devrait jamais être possible. Si c'est le cas, le site stocke les mots de passe en clair ou de façon réversible.
	- **Envoi d'un nouveau mot de passe temporaire** : à éviter. Si ce mot de passe n'expire pas vite ou si l'utilisateur ne le change pas immédiatement, il est exposé aux attaques de l'homme du milieu. L'e-mail n'est pas un canal sûr : les boîtes de réception sont persistantes, mal adaptées au stockage d'informations confidentielles et souvent synchronisées sur plusieurs appareils.
	- **Envoi d'une URL unique vers une page de réinitialisation** : méthode la plus sûre, à condition d'être bien implémentée.
- Implémentation faible : l'URL contient un paramètre prévisible.
  `http://vulnerable-website.com/reset-password?user=victim-user`
  Si ce paramètre est modifiable, l'attaquant remplace le nom d'utilisateur et accède à la page de réinitialisation du compte visé sans jamais avoir reçu le lien.
- Meilleure implémentation : un jeton à forte entropie dans l'URL, qui ne donne aucun indice sur l'utilisateur concerné. Le serveur doit :
	- retrouver l'utilisateur associé au jeton côté serveur ;
	- faire expirer le jeton rapidement ;
	- le détruire dès que le mot de passe a été changé.
- Certains sites ne revalident pas le jeton à la soumission du formulaire. L'attaquant ouvre alors le formulaire avec son propre jeton, le supprime de la requête, puis réinitialise le mot de passe de n'importe quel utilisateur.
- Si l'URL de l'e-mail est générée dynamiquement (à partir de l'en-tête `Host` ou `X-Forwarded-Host`), elle peut être vulnérable au **password reset poisoning** : l'attaquant fait pointer le lien vers son propre domaine et récupère le jeton de la victime quand elle clique.

### Modifier le mot de passe
- Le formulaire demande généralement le mot de passe actuel, puis le nouveau mot de passe deux fois.
- Il repose sur le même mécanisme de vérification qu'une page de connexion classique, et est donc exposé aux mêmes attaques (force brute, énumération).
- Cette page devient particulièrement dangereuse quand l'attaquant peut y accéder sans être connecté en tant que sa victime.
- Cas typique : le nom d'utilisateur est transmis dans un champ masqué du formulaire. L'attaquant modifie cette valeur dans la requête pour cibler un utilisateur arbitraire.

## Comment détecter
- Cookie persistant : se connecter avec "Se souvenir de moi", puis comparer les cookies obtenus avec plusieurs comptes (ou au fil du temps) pour repérer une structure (nom d'utilisateur, horodatage, base64, hachage connu).
- Réinitialisation : lancer la procédure avec son propre compte et observer :
	- le contenu du lien reçu (paramètre `user` prévisible ou jeton aléatoire) ;
	- si le jeton est encore contrôlé à la soumission du formulaire (le supprimer ou le modifier dans la requête) ;
	- si l'en-tête `X-Forwarded-Host` ou `Host` influence le domaine du lien généré.
- Modification du mot de passe : vérifier si le nom d'utilisateur figure dans un champ masqué, et si les messages d'erreur diffèrent selon que le mot de passe actuel est correct ou non.

## Comment exploiter (principe)
- Cookie persistant : forger le cookie d'une victime à partir de la structure déduite, ou casser le hachage qu'il contient.
- Réinitialisation avec paramètre prévisible : remplacer le nom d'utilisateur dans l'URL.
- Jeton non revalidé : retirer le jeton de la requête de soumission.
- Password reset poisoning : envoyer la demande de réinitialisation de la victime avec un en-tête `X-Forwarded-Host` pointant vers le serveur de l'attaquant, puis lire le jeton dans les journaux d'accès à la réception du clic de la victime, et l'utiliser sur le vrai lien de réinitialisation.
- Modification du mot de passe : changer le champ `username` pour viser la victime, puis exploiter la différence de message d'erreur pour trouver son mot de passe actuel par force brute.

## Pièges et points d'attention BSCP
- Dans la page de modification du mot de passe, le verrouillage de compte ne se déclenche que si les deux nouveaux mots de passe sont identiques. Avec deux valeurs différentes, la page répond `Current password is incorrect` ou `New passwords do not match`, ce qui permet de tester des mots de passe sans verrouiller le compte.
- Pour le password reset poisoning, le lien à utiliser est celui reçu dans votre propre boîte e-mail (pas celui qui pointe vers le serveur d'exploit), dans lequel vous remplacez uniquement la valeur du jeton.
- Si `X-Forwarded-Host` est ignoré, essayer `Host` ou d'autres en-têtes de redirection (`X-Forwarded-Server`, `X-Host`, `Forwarded`).

## Prévention
- Plusieurs principes permettent de réduire le risque lié à la gestion de l'authentification :
	- **Protéger les identifiants des utilisateurs**
		- Ne jamais transmettre de données de connexion sur une connexion non chiffrée : toujours utiliser HTTPS.
		- Vérifier qu'aucun nom d'utilisateur ni adresse e-mail n'est exposé, que ce soit par des profils publics ou par des réponses HTTP qui les reflètent.
	- **Ne pas compter sur les utilisateurs pour assurer la sécurité**
		- Une authentification stricte demande un effort aux utilisateurs, qui chercheront à l'éviter.
		- Il faut donc imposer les comportements sécurisés.
	- **Politique de mot de passe**
		- Les politiques traditionnelles échouent souvent : les utilisateurs adaptent leurs mots de passe prévisibles aux règles imposées.
		- Une alternative plus efficace est un vérificateur de mot de passe qui évalue la solidité en temps réel pendant la saisie.
		- N'autoriser que les m