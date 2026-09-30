## En bref
- Méthode de sécurité qui demande à un utilisateur de fournir 2 preuves de qui il est afin d'accéder à son compte ou à une application.

## Types et variantes
- Two-factor authentification tokens
	- Généralement lu par un utilisateur à partir d'un appareil physique ou un application comme le Microsoft Authenticator.
	- Possibilité de recevoir un SMS mais il y a un risque de "Sim swapping", c'est le fait de s'emparer de la carte SIM de la victime.
## Comment détecter
- 

## Comment exploiter (principe)
- 

## Pièges et points d'attention BSCP
- Si l’utilisateur est d’abord invité à saisir un mot de passe, puis à entrer un code de vérification sur une page distincte, il se trouve en réalité dans un état « connecté » avant même d’avoir saisi ce code.
- Dans ce cas, il vaut la peine de tester si vous pouvez accéder directement aux pages « réservées aux utilisateurs connectés » après avoir franchi la première étape d’authentification.
- Logique défaillante dans l'authentification à deux facteurs empêche le site web de vérifier correctement, une fois que l'utilisateur a terminé la première étape de connexion, que c'est bien le même utilisateur qui effectue la deuxième étape.
- Par exemple, l'utilisateur se connecte avec ses identifiants habituels lors de la première étape, comme suit :
```
POST /login-steps/first HTTP/1.1
Host: vulnerable-website.com
...
username=carlos&password=qwerty
```
- Un cookie associé à son compte lui est alors attribué, avant qu'il ne soit redirigé vers la deuxième étape du processus de connexion :
```
HTTP/1.1 200 OK

Set-Cookie: account=carlos
  

GET /login-steps/second HTTP/1.1

Cookie: account=carlos
```
- Lors de l’envoi du code de vérification, la requête utilise ce cookie pour déterminer à quel compte l’utilisateur tente d’accéder :
```
POST /login-steps/second HTTP/1.1

Host: vulnerable-website.com

Cookie : account=carlos
...
verification-code=123456
```
- Dans ce cas, un attaquant pourrait se connecter en utilisant ses propres identifiants, puis modifier la valeur du cookie « account » pour lui attribuer n’importe quel nom d’utilisateur arbitraire.
```
POST /login-steps/second HTTP/1.1
Host : vulnerable-website.com
Cookie : account=victim-user
...
verification-code=123456
```
- Cela s'avère extrêmement dangereux si l'attaquant est ensuite capable de deviner le code de vérification par force brute, car cela lui permettrait de se connecter aux comptes d'utilisateurs arbitraires en se basant uniquement sur leur nom d'utilisateur.Il n'aurait même pas besoin de connaître le mot de passe de l'utilisateur.
## Prévention
- 

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
- 

## Mes notes