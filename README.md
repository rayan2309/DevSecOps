# Rendu des exercices DevSecOps - ARROUD Rayan

# Exercice 1 – Détection de secrets avec Gitleaks

## Contexte

Une équipe de développement demande un audit du dépôt Git **OWASP/wrongsecrets** afin de vérifier qu'aucun secret n'a été publié accidentellement.

**Dépôt audité :** https://github.com/OWASP/wrongsecrets

---

## 1. Cloner le dépôt

```
git clone https://github.com/OWASP/wrongsecrets
```

!Capture d'écran 2026-06-15 105516.png

---

Le dépôt contient 44 304 objets pour environ 168 Mo.

## 2. Installer Gitleaks

Installation via **winget** sous Windows :

```powershell
winget install gitleaks
```

---

## 3. Scanner le dépôt

Deux scans ont été réalisés :

**Scan de l'historique Git complet** (détecte les secrets même supprimés depuis) :

```
cd wrongsecrets
gitleaks detect --source . --verbose --report-path gitleaks-report.json
```

!Capture d'écran 2026-06-15 110316.png

**Scan sans historique** (uniquement les fichiers présents) :

```
gitleaks detect --source . --no-git
```

Résultat : **1 879 leaks** détectés sur les fichiers actuels.

!Capture d'écran 2026-06-15 110446.png

**Scan avec export JSON** pour analyse détaillée :

```
gitleaks detect --source . -v -f json -r rapport.json

```

---

## 4. Identifier les secrets détectés

### Nombre total de secrets

```powershell
(Get-Content rapport.json | ConvertFrom-Json).Count
```

### Répartition par type

```powershell
(Get-Content rapport.json | ConvertFrom-Json) | Group-Object RuleID | Sort-Object Count -Descending | Select-Object Count, Name
```

!Capture d'écran 2026-06-15 110954.png

**Résultat : 1 043 secrets détectés** dans l'historique Git.

| Nombre | Type de secret |
| --- | --- |
| 970 | generic-api-key |
| 20 | slack-webhook-url |
| 18 | private-key |
| 9 | kubernetes-secret-yaml |
| 9 | slack-bot-token |
| 5 | aws-access-token |
| 4 | vault-service-token |
| 2 | jwt |
| 2 | gcp-api-key |
| 1 | github-pat |
| 1 | gitlab-pat |
| 1 | github-fine-grained-pat |
| 1 | hashicorp-tf-api-token |

---

## 5. Niveau de criticité

Gitleaks ne fournit pas de score de criticité automatique. L'analyse suivante est basée sur l'impact potentiel de chaque type de secret s'il était réel et actif.

### Critique – Accès direct à l'infrastructure

| Type | Nombre | Risque |
| --- | --- | --- |
| aws-access-token | 5 | Accès complet à un compte AWS (machines, stockage S3, bases de données) |
| private-key | 18 | Accès direct aux serveurs via SSH, déchiffrement de communications |
| gcp-api-key | 2 | Accès aux services Google Cloud Platform |
| vault-service-token | 4 | Accès à HashiCorp Vault, compromission de tous les secrets qu'il protège |

### Élevée – Accès à des services tiers

| Type | Nombre | Risque |
| --- | --- | --- |
| slack-webhook-url | 20 | Envoi de messages dans des canaux Slack, phishing interne |
| slack-bot-token | 9 | Lecture/écriture des messages, fichiers et canaux privés Slack |
| github-pat | 1 | Accès en lecture/écriture aux dépôts Git, code source, repos privés |
| gitlab-pat | 1 | Idem sur GitLab |
| github-fine-grained-pat | 1 | Token GitHub à permissions granulaires |
| jwt | 2 | Usurpation d'identité d'un utilisateur authentifié |
| hashicorp-tf-api-token | 1 | Modification ou destruction de l'infrastructure via Terraform Cloud |

### Moyenne – À examiner au cas par cas

| Type | Nombre | Risque |
| --- | --- | --- |
| kubernetes-secret-yaml | 9 | Secrets Kubernetes en clair (mots de passe, certificats, tokens de service) |
| generic-api-key | 970 | Catégorie générique : chaque occurrence doit être examinée individuellement |

---

## 6. Actions correctives

### Remédiation immédiate

- **Révoquer et regénérer** tous les secrets réels exposés. Un secret commité dans Git est considéré comme compromis, même s'il a été supprimé ensuite, car il reste accessible dans l'historique des commits.
- **Purger l'historique Git** avec des outils comme `git filter-repo` ou BFG Repo-Cleaner pour supprimer définitivement les commits contenant des secrets.

### Externalisation des secrets

- Ne jamais stocker de secret en dur dans le code source.
- Utiliser un gestionnaire de secrets : HashiCorp Vault, AWS Secrets Manager, Azure Key Vault.
- Stocker les secrets de développement local dans des fichiers `.env` et les ajouter au `.gitignore`.

### Prévention automatisée

- Installer un **hook pre-commit Gitleaks** pour bloquer tout commit contenant un secret :

```bash
gitleaks protect --staged
```

- Intégrer Gitleaks dans la **pipeline CI/CD** (GitHub Actions, GitLab CI) pour scanner automatiquement chaque push et merge request.
- Mettre en place un `.gitignore` strict :

```
.env
*.pem
*.key
*.p12
*credentials*
```

- Former les développeurs à la gestion sécurisée des secrets.

---

### Combien de secrets ont été détectés ?

**1 043 secrets** ont été détectés dans l'historique Git, répartis en 13 catégories distinctes. Un scan des fichiers actuels (sans historique) révèle 1 879 leaks.

### Quels types de secrets et quels risques représentent-ils ?

Les secrets les plus critiques sont les **clés d'accès cloud** (AWS, GCP), les **clés privées** (RSA/SSH) et les **tokens Vault**, qui permettent un accès direct à l'infrastructure. Viennent ensuite les **tokens d'API** (Slack, GitHub, GitLab), les **JWT** et les **webhooks**, qui donnent accès à des services tiers. La majorité des findings (970) sont des **generic-api-key** nécessitant un tri manuel. On trouve aussi des **secrets Kubernetes** en clair.

### Comment éviter leur présence dans Git ?

Trois axes complémentaires : externaliser les secrets via un gestionnaire dédié et des variables d'environnement, automatiser la détection avec un hook pre-commit Gitleaks et une intégration CI/CD, et sensibiliser les développeurs aux bonnes pratiques (`.gitignore` strict, revue de code, politique de rotation des secrets).

# Exercice 2 – Analyse statique de code (SAST) avec Semgrep
