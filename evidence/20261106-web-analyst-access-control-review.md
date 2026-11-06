# Revue documentaire de la note Access-control

Date : 2026-11-06
Note revue : /home/capss/web-bscp/BSCP-Vault/01-Server-side/Access-control.md (213 lignes, statut "en cours")
Méthode : lecture de la note parente et des en-têtes des six sous-notes (toutes au statut "à faire", repérées seulement pour les doublons et les liens). Aucune requête envoyée, rien rejoué, aucune note modifiée.
Convention : tout ce qui est marqué "à vérifier" n'a pas été rejoué par l'utilisateur. Le catalogue vient de connaissances générales sur la classe de vulnérabilités, pas d'un test sur une cible.

## 0. État des labs d'après le contenu de la note

Indices dans la note :
- Ligne 68 : case Apprentice cochée avec la date 2026-09-17. Practitioner et Expert non cochés.
- Lignes 74-75 : l'encadré d'avertissement dit que le journal a été reconstitué de mémoire à partir de solutions publiques, sans rejeu. Aucune ligne du journal n'est donc un journal écrit par l'utilisateur à partir de son propre rejeu.

Classement :

| Labs | Niveau | État d'après la note | Recommandation |
| --- | --- | --- | --- |
| 1 à 9 | Apprentice | Réalisés selon la case cochée du 2026-09-17, mais journal non issu du rejeu (encadré d'avertissement) | L'utilisateur confirme et reformule à sa façon ce qu'il a réellement observé (voir plan de réécriture, étape 4). |
| 10 à 13 | Practitioner | Non réalisés (case non cochée) | Laisser la note dans son état actuel. Aucune solution, aucun déroulé, aucun plan de résolution proposé dans cette revue. |
| Expert | Expert | Aucun lab listé, case non cochée | Rien à faire. |

Point d'attention : les labs 10 à 13 contiennent déjà des étapes détaillées copiées de solutions publiques. C'est une information que l'utilisateur voudra peut-être ne pas avoir sous les yeux avant de faire les labs. La recommandation de ce rapport est de ne pas y toucher, la décision de les masquer ou non lui revient.

Dans le catalogue de la partie 3, les variantes liées aux labs 10 à 13 sont décrites uniquement comme des familles de techniques génériques (concept, signal, limites), sans lien avec un lab précis et sans suite d'étapes.

## 1. Ce qui manque ou est flou

Concepts
- Le contrôle d'accès n'est pas décomposé en trois modèles (vertical, horizontal, dépendant du contexte). Le contrôle d'accès dépendant du contexte (état de l'application, ordre des étapes) est seulement suggéré par le multi-étapes. À nommer explicitement.
- Aucune distinction entre modèles de contrôle (rôles, listes de contrôle d'accès, attributs) ni entre contrôle programmatique et déclaratif. Utile pour comprendre pourquoi les contournements de plateforme existent.
- La notion d'objet de référence directe est absente de l'introduction : IDOR n'est défini que dans la sous-note. Développer l'acronyme à la première occurrence (ligne 20).
- Pas d'explication de pourquoi la réécriture d'URL par en-tête fonctionne (le front-end filtre sur la ligne de requête, le back-end route selon l'en-tête). Ligne 56 le dit en une phrase, sans le mécanisme.

Méthodologie de détection
- Aucune stratégie de comparaison à deux sessions (utilisateur A et utilisateur B) avec rejeu automatique. Mentionner les extensions Burp usuelles de comparaison de droits (Autorize, AuthMatrix) ou la fonction de rejeu avec une autre session. Citer le nom exact et vérifier la disponibilité avant de les ajouter.
- Pas de liste de signaux de réussite ou d'échec : code de statut, longueur de réponse, corps de redirection non vide, message d'erreur différent. Le cas "302 avec corps contenant la donnée" (lab 7) n'est présenté que dans le journal, pas comme règle de détection.
- Pas de méthode pour énumérer les identifiants (Intruder, séquences, fuites ailleurs dans l'application) ni pour détecter un contrôle côté client (rôle présent dans le JavaScript ou dans un jeton).
- Les lignes 38 et 53 répètent la même consigne sur les méthodes HTTP sans dire quoi observer (405, 403, 200 différent).

Cas par variante absents ou à peine abordés
- Variantes de chemin : casse, barre oblique finale, extension, doubles barres, encodage, point-virgule de paramètre de chemin. Seul "variantes de chemin" apparaît ligne 44, sans exemple.
- Autres en-têtes de réécriture : `X-Rewrite-URL` à côté de `X-Original-URL`.
- Contournement de limitation d'adresse source par en-têtes de transfert (`X-Forwarded-For`, etc.) : absent.
- Surcharge de méthode (`X-HTTP-Method-Override`) : absent, alors que la section méthodes HTTP est un thème central.
- Pollution et duplication de paramètres, remplacement du format de contenu (JSON, formulaire) pour atteindre un champ protégé : seul le cas du champ de rôle ajouté au corps est évoqué (mass assignment), sans le nommer.
- Identifiants dans le corps, les en-têtes, les cookies, pas seulement dans la chaîne de requête.
- Jetons et JWT portant un rôle : renvoi vers une note dédiée absent.
- API (REST, GraphQL) : propriétés d'objet exposées, listes d'objets sans filtre : absent.
- Escalade verticale via fonction de réinitialisation de mot de passe ou changement d'e-mail d'un autre compte : absent.

Limites
- Rien sur ce qui empêche l'exploitation : jeton anti-falsification, liaison de session, limitation de débit, contrôles asynchrones, réponses uniformes.
- Pas de rappel que la preuve doit établir l'accès réel à la donnée d'un autre compte, pas seulement un changement de code de statut.

Contre-mesures
- La section Prévention est correcte mais générale. Manquent : journalisation et alerte sur les refus, tests automatisés d'autorisation, désactivation des en-têtes de réécriture côté serveur d'application, ne pas faire confiance à `Referer`, vérification de propriété au niveau de la couche de données, éviter de renvoyer des données dans le corps d'une redirection ou d'un message d'erreur. L'idée sur la redirection et le pré-remplissage de mot de passe (labs 7 et 8) n'a pas de contre-mesure associée.

Pièges BSCP
- Pas de mention du temps limité de l'examen et de la priorisation : le contrôle d'accès est typiquement l'étape de l'accès à l'administration et du pivot vers un compte privilégié. Documenter le rôle de cette classe dans l'enchaînement (accès initial, puis suite de la chaîne), à confirmer d'après le programme officiel de l'examen.
- Lister les artefacts à repérer systématiquement en phase de cartographie : `robots.txt`, JavaScript, commentaires, panneaux d'administration, réponses JSON portant un champ de rôle.
- Penser à regarder les en-têtes et cookies après connexion avec chaque compte fourni, et à ne pas oublier la fonction de changement d'e-mail, de mot de passe ou de profil comme point d'entrée de champs cachés.

## 2. Ce qui est superflu, redondant ou hors sujet

Remarques de rédaction laissées dans le texte
- Lignes 74-75 : l'encadré "Étapes rédigées de mémoire" est une remarque de travail, pas un contenu de note. Elle devra disparaître une fois le journal réécrit à partir de vrais rejeux. Pour les labs 10 à 13 non réalisés, la laisser telle quelle (voir partie 0).
- Ligne 200 : section "Mes notes" avec une puce vide.

Redondances
- Lignes 38 et 53 : même consigne sur les méthodes HTTP, dupliquée.
- Lignes 21 et 54 : la bascule horizontale vers verticale est dite deux fois, et une troisième fois dans la sous-note escalade-horizontale.
- Lignes 56 et 44 : la réécriture d'URL et les méthodes alternatives sont répétées entre détection, exploitation et pièges.
- Section "Comment exploiter (principe)" (lignes 40-47) recoupe la section "Comment détecter" (lignes 32-38) : les étapes 1 et 2 répètent la cartographie et la comparaison de rôles. Fusionner en une section "Méthodologie".
- Ligne 50 (trois cas) et ligne 34 (matrice de droits) disent la même chose.
- Ligne 64 et ligne 51 : la sécurité par l'obscurité apparaît dans les pièges et dans la prévention.
- La table des types (lignes 23-30) et la liste de liens (lignes 203-208) répètent les mêmes six liens. Les six sont déjà dans la table.
- Le champ du niveau apparaît à la fois dans le frontmatter (`niveau: practitioner`) et dans la table.
- Ligne 20 : IDOR est cité comme "très proche" de l'escalade horizontale, puis traité comme type séparé dans la table : clarifier la relation (IDOR comme cas particulier de l'escalade horizontale, comme le dit la sous-note idor).

Liens hors sujet ou à justifier
- `[[Authentication]]` : lié à la distinction authentification et autorisation (ligne 55). Défendable, mais à rattacher dans le texte plutôt que dans la liste finale.
- `[[Business-logic]]` : défendable pour le multi-étapes. À rattacher au cas concerné.
- `[[Information-disclosure]]` : défendable pour les fuites (labs 6, 7, 8). À rattacher dans le texte.
- `[[Path-traversal]]` : peu lié. Aucune occurrence dans le texte qui le justifie. Candidat à la suppression, ou à garder en "voir aussi" avec une phrase de justification.
- `[[Payloads-cheatsheet]]` : pertinent, mais le texte ne dit pas quels éléments y renvoient. Après report de la partie 4, ajouter une phrase.
- Image `![](Access-control.png)` ligne 14 : fichier non vérifié dans cette revue (à vérifier, existence et pertinence).

Style
- Emoji dans la ligne 68 (coche verte) : non conforme à la convention de style des livrables (à confirmer pour les notes du coffre).
- Abréviations : "UI" ligne 53, "IDOR" et "GUID" jamais développés. À écrire en toutes lettres à la première occurrence.
- Mélange d'anglais et de français sans règle : "front-end", "deny by default" (ligne 62, déjà traduit), "Repeater" (outil, acceptable).
- Aucune faute d'orthographe bloquante repérée. Constat de cohérence : les labs 3 à 8 désignent l'élément à modifier tantôt "cookie", tantôt "paramètre", tantôt "valeur" ; harmoniser.

## 3. Catalogue de payloads, requêtes et manipulations

Légende : tout est "à vérifier" (non rejoué par l'utilisateur). Les éléments sont génériques. Pour les familles liées à des labs non réalisés (contournement de plateforme, multi-étapes, Referer), seules les techniques sont décrites, pas de procédure de résolution.

### 3.1 Fonctionnalité non protégée
| Manipulation | Contexte d'usage | Signal attendu | Limites |
| --- | --- | --- | --- |
| Lecture de `/robots.txt` et `/sitemap.xml` | Cartographie initiale | Chemins d'administration listés (`Disallow`) | Le fichier peut être vide ou leurre. À vérifier |
| Recherche dans le JavaScript et les commentaires HTML de chemins `admin`, `internal`, `manage` | URL non devinable divulguée côté client | Chemin dans une affectation d'attribut ou une route | Fichiers minifiés : utiliser une recherche plein texte. À vérifier |
| Énumération de chemins courants (`/admin`, `/administrator`, `/manage`, `/console`) avec un outil de découverte de contenu | Aucun lien visible | Code 200 ou 403 différent de 404 | Bruit et limitation de débit. À vérifier |
| Accès direct sans session, puis avec session de faible privilège | Chaque page sensible | 200 avec contenu d'administration | Une page peut s'afficher sans que les actions soient autorisées : tester l'action elle-même. À vérifier |

### 3.2 Paramètre contrôlé par l'utilisateur et affectation de masse
| Manipulation | Contexte | Signal | Limites |
| --- | --- | --- | --- |
| Cookie `Admin=false` vers `Admin=true` | Cookie en clair portant un drapeau | Page ou fonction d'administration accessible | Si le cookie est signé ou chiffré, l'altération échoue. À vérifier |
| Paramètre de requête ou champ caché `admin=true`, `role=admin`, `isAdmin=1` | Formulaires, URL | Changement de contenu ou de droits | Dépend du nom exact du paramètre. À vérifier |
| Ajout d'un champ `"roleid":2` (ou `"role":"admin"`) dans le corps JSON d'une mise à jour de profil | Réponse JSON qui montre déjà le champ | Champ accepté, rôle changé, accès à la fonction | Le serveur peut ignorer les champs inconnus. À vérifier |
| Changer le format de contenu (JSON vers formulaire) pour contourner un filtre de champs | Filtrage par format | Champ pris en compte | Dépend de l'application. À vérifier |

### 3.3 Escalade horizontale et IDOR
| Manipulation | Contexte | Signal | Limites |
| --- | --- | --- | --- |
| `?id=wiener` vers `?id=carlos` | Identifiant lisible | Données de l'autre utilisateur | Contrôle de propriété côté serveur l'empêche. À vérifier |
| Identifiant numérique incrémenté ou décrémenté (`1.txt`, `2.txt`) | Fichiers statiques à nom séquentiel | Contenu d'un autre utilisateur | Nécessite un compte qui génère l'objet pour comprendre le schéma. À vérifier |
| Recherche du GUID d'un utilisateur dans les pages publiques (articles, commentaires, profils) | Identifiants imprévisibles mais exposés | GUID de la cible trouvé, accès aux données | Si le GUID n'est exposé nulle part, impasse. À vérifier |
| Lecture du corps d'une redirection (`302`) | La redirection est déclenchée par l'absence d'autorisation | Corps non vide contenant la donnée | Navigateur masque le corps : utiliser Burp. À vérifier |
| Identifiant dans le corps, un en-tête ou un cookie plutôt que l'URL | API | Réponse d'un autre utilisateur | Plus d'endroits à tester. À vérifier |
| Remplacement de l'identifiant par celui d'un compte administrateur (`id=administrator`) | Fuite de champs sensibles (mot de passe pré-rempli) | Valeur du champ dans le HTML | La donnée peut être masquée visuellement mais présente dans le source. À vérifier |
| Séquence d'identifiants avec Intruder | Identifiants numériques | Longueur de réponse différente | Limitation de débit. À vérifier |

### 3.4 Contournement lié à la plateforme (techniques génériques)
| Manipulation | Contexte | Signal | Limites |
| --- | --- | --- | --- |
| En-tête `X-Original-URL: /chemin` sur une requête vers `/` | Un front-end filtre l'URL, le back-end lit l'en-tête | Réponse correspondant à `/chemin` | Dépend du framework ou du proxy. À vérifier |
| En-tête `X-Rewrite-URL: /chemin` | Même principe, autre implémentation | Idem | Idem. À vérifier |
| Test de reconnaissance : `X-Original-URL: /chemin-inexistant` | Vérifier que l'en-tête est pris en compte | Réponse "not found" venant de l'application | Ne prouve que la prise en compte, pas l'accès. À vérifier |
| Paramètres de requête conservés dans la requête réelle, chemin dans l'en-tête | Actions avec paramètres | Action exécutée | Selon la gestion des paramètres. À vérifier |
| Changement de méthode (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`) | Règle de contrôle liée à une méthode | Passage de 401/403 à 200 | Parfois la méthode alternative exige un autre format de paramètre. À vérifier |
| `X-HTTP-Method-Override: PUT` (ou `_method=PUT`) sur une requête `POST` | Frameworks qui acceptent la surcharge | Comportement de la méthode surchargée | Souvent désactivé. À vérifier |
| Variantes de chemin : `/Admin`, `/admin/`, `/admin//`, `/admin.json`, `/admin;x`, `/%61dmin`, `/./admin` | Règles de filtrage par correspondance exacte | Passage du refus à 200 | Dépend de la normalisation. À vérifier |
| Faux en-têtes de provenance (`X-Forwarded-For: 127.0.0.1`) | Règle liée à l'adresse source | Accès accordé | Généralement ignoré si non configuré. À vérifier |

### 3.5 Multi-étapes et Referer (techniques génériques)
| Manipulation | Contexte | Signal | Limites |
| --- | --- | --- | --- |
| Rejeu direct d'une étape tardive en omettant les précédentes | Seule la première étape est protégée | Action exécutée | Jetons d'état par étape l'empêchent. À vérifier |
| Remplacement de la session par celle d'un compte moins privilégié sur une requête sensible | Mesurer quelles étapes vérifient le rôle | Refus ou succès par étape | Utiliser deux sessions distinctes. À vérifier |
| Présence, absence et modification de l'en-tête `Referer` | Contrôle fondé sur la provenance | Refus sans l'en-tête, succès avec une valeur attendue | Le serveur peut contrôler l'origine complète. À vérifier |

### 3.6 Jetons, API et divers
| Manipulation | Contexte | Signal | Limites |
| --- | --- | --- | --- |
| Décoder et examiner un jeton de session (base64, JWT) à la recherche d'un rôle | Rôle encodé côté client | Champ de rôle lisible | Signature empêche la modification (voir la note dédiée si elle existe). À vérifier |
| Lister les propriétés d'un objet dans une réponse JSON et les renvoyer en écriture | API REST | Propriété acceptée | Liste blanche de champs. À vérifier |
| Comparer les réponses entre trois contextes : sans session, utilisateur standard, administrateur | Toute fonction sensible | Différence de statut ou de longueur | Réponses uniformes peuvent masquer l'écart. À vérifier |

## 4. À reporter dans la cheat sheet

Destination : /home/capss/web-bscp/BSCP-Vault/04-Essential-skills/Payloads-cheatsheet.md (non modifié par cette revue). Tout est "à vérifier" tant que l'utilisateur ne l'a pas rejoué.

```
# Access control - courts, à vérifier
Cookie: Admin=true
{"roleid":2}                               (ajout de champ dans le JSON de profil)
?id=<autre utilisateur>                     (escalade horizontale)
/download-transcript/1.txt                  (identifiant séquentiel de fichier)
GET /  + X-Original-URL: /admin             (réécriture d'URL)
GET /  + X-Rewrite-URL: /admin              (variante)
X-Original-URL: /chemin-inexistant          (test de prise en compte de l'en-tête)
X-HTTP-Method-Override: PUT                 (surcharge de méthode)
Changer la méthode : GET, POST, PUT, PATCH, DELETE, HEAD
/Admin   /admin/   /admin//   /admin.json   /admin;x   /%61dmin   /./admin
X-Forwarded-For: 127.0.0.1                  (restriction par adresse source)
Referer: <URL de la page d'administration>  (contrôle fondé sur la provenance)
Lire le corps des réponses 302 (donnée dans le corps de la redirection)
```

## 5. Plan de réécriture de la note

1. Frontmatter : conserver le niveau et le statut ; ajouter un champ pour la date de dernière mise à jour. Retirer la coche emoji du suivi des labs au profit d'une date en texte.
2. Introduction : réduire "En bref" à quatre puces, développer IDOR et GUID à la première occurrence, ajouter les trois modèles (vertical, horizontal, dépendant du contexte).
3. Fusionner "Comment détecter" et "Comment exploiter (principe)" en une seule section "Méthodologie", structurée par variante (une sous-section par ligne de la table des types), avec pour chacune : signal de réussite, signal d'échec, limites.
4. Pièges BSCP : dédupliquer les lignes 21/54, 38/53 et 51/64, et ajouter les pièges de la partie 1 (étendue réelle de la preuve, corps de redirection, cartographie systématique).
5. Prévention : ajouter les contre-mesures manquantes (journalisation, tests d'autorisation, désactivation des en-têtes de réécriture, pas de donnée dans les redirections).
6. Table des types : conserver, supprimer la liste de liens finale redondante, ne garder en "Voir aussi" que les liens justifiés dans le texte (Authentication, Business-logic, Information-disclosure, Payloads-cheatsheet). Retirer ou justifier Path-traversal.
7. Journal des labs, Apprentice (labs 1 à 9) : l'utilisateur dit s'il les a bien réalisés. Si oui, il remplace chaque entrée par ses propres observations (requête réelle, signal, piège rencontré), puis retire l'encadré d'avertissement pour ces labs.
8. Journal des labs, Practitioner (labs 10 à 13) : non réalisés, laisser en l'état. L'utilisateur décidera lui-même s'il masque les étapes actuelles avant de faire les labs. Aucune solution ni plan n'est proposé ici.
9. Supprimer la section "Mes notes" vide, ou la remplir après les labs.
10. Reporter les lignes de la partie 4 dans la cheat sheet, puis ajouter une phrase de renvoi dans la note.
11. Vérifier que l'image Access-control.png existe et correspond au contenu.
