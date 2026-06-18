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

<<<<<<< HEAD
![Clone du dépôt](images/exo1/Capture%20d'écran%202026-06-15%20105516.png)
=======
!(images/exo1/Capture d'écran 2026-06-15 105516.png)
>>>>>>> 6213621efd7ea116bc75528c205f8f3007717375

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

<<<<<<< HEAD
![Scan historique Git](images/exo1/Capture%20d'écran%202026-06-15%20110316.png)
=======
!(images/exo1/Capture d'écran 2026-06-15 110316.png)
>>>>>>> 6213621efd7ea116bc75528c205f8f3007717375

**Scan sans historique** (uniquement les fichiers présents) :

```
gitleaks detect --source . --no-git
```

Résultat : **1 879 leaks** détectés sur les fichiers actuels.

<<<<<<< HEAD
![Scan no-git 1879 leaks](images/exo1/Capture%20d'écran%202026-06-15%20110446.png)
=======
!(images/exo1/Capture d'écran 2026-06-15 110446.png)
>>>>>>> 6213621efd7ea116bc75528c205f8f3007717375

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

<<<<<<< HEAD
![Répartition par type](images/exo1/Capture%20d'écran%202026-06-15%20110954.png)
=======
!(images/exo1/Capture d'écran 2026-06-15 110954.png)
>>>>>>> 6213621efd7ea116bc75528c205f8f3007717375

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

## Contexte

Une API Python développée rapidement doit être auditée avant sa mise en production.

**Dépôt audité :** https://github.com/trottomv/python-insecure-app

---

## 1. Cloner le dépôt

```
git clone https://github.com/trottomv/python-insecure-app
cd python-insecure-app
```

![Clone du dépôt](images/exo2/Capture%20d'écran%202026-06-15%20120159.png)

---

## 2. Installer Semgrep

Installation via pip sous Windows :

```powershell
pip install semgrep
```

---

## 3. Lancer une analyse

Plusieurs scans ont été réalisés avec des rulesets progressivement élargis.

**Premier scan (règles auto) :**

```
semgrep scan --config=auto --json -o semgrep-report.json .
```

![Résultat scan auto](images/exo2/Capture%20d'écran%202026-06-15%20154142.png)

**Scan élargi avec les packs Python, OWASP Top 10 et security-audit :**

```
semgrep scan --config "p/python" --config "p/owasp-top-ten" --config "p/security-audit" . --json -o semgrep-report.json
```

![Scan élargi](images/exo2/Capture%20d'écran%202026-06-15%20154603.png)

---

## 4. Analyser les résultats

```powershell
(Get-Content semgrep-report.json | ConvertFrom-Json).results.Count
```

![Résultat 0 findings](images/exo2/Capture%20d'écran%202026-06-18%20103125.png)

Malgré 678 règles appliquées sur 25 fichiers, Semgrep remonte **0 finding**.

---

## 5. Identifier les vulnérabilités détectées

Semgrep n'a détecté aucune vulnérabilité automatiquement. 


### Quelles vulnérabilités ont été identifiées ?

Trois vulnérabilités ont été identifiées par analyse manuelle : une **injection de template côté serveur (SSTI)** dans `app/main.py:41`, une **faille XSS** dans `app/main.py:38`, et des **secrets hardcodés** dans `app/config.py:13-15`.

### Quels risques présentent-elles ?

Le SSTI est la vulnérabilité la plus critique : il permet à un attaquant d'exécuter du code arbitraire sur le serveur (RCE), pouvant mener à une compromission totale de la machine. Le XSS permet l'injection de JavaScript malveillant dans le navigateur des utilisateurs (vol de session, phishing). Les secrets hardcodés exposent des credentials dans l'historique Git, accessibles à quiconque clone le dépôt.

### Les résultats vous semblent-ils tous pertinents ?

Non. Semgrep a retourné **0 finding** malgré l'application de 678 règles issues de trois rulesets différents (p/python, p/owasp-top-ten, p/security-audit). Il s'agit de **faux négatifs** : l'outil n'a pas été capable de détecter les vulnérabilités réelles présentes dans le code. Cela illustre la limite des outils SAST automatiques, qui ne remplacent pas une revue manuelle du code, notamment pour des patterns complexes comme la construction dynamique de templates.

### Quelles corrections recommanderiez-vous ?

Remplacer la construction dynamique du template par un template statique avec variables Jinja2 (neutralise SSTI et XSS simultanément), externaliser tous les secrets dans des variables d'environnement stockées hors du dépôt, et intégrer un outil de détection de secrets comme Gitleaks en hook pre-commit pour éviter tout futur commit de credentials.

---

# Exercice 3 – Analyse des dépendances (SCA) avec Trivy

## Contexte

Une application Java Spring Boot utilise de nombreuses bibliothèques Open Source. L'objectif est de vérifier si certaines contiennent des vulnérabilités connues avant toute mise en production.

**Dépôt audité :** https://github.com/veracode/verademo

---

## 1. Cloner le dépôt

```
git clone https://github.com/veracode/verademo
cd verademo
```

![Clone du dépôt](images/exo3/Capture%20d'écran%202026-06-18%20105932.png)

---

## 2. Installer Trivy

```powershell
winget install AquaSecurity.Trivy
```

![Installation Trivy](images/exo3/Capture%20d'écran%202026-06-18%20110205.png)

---

## 3. Scanner le système de fichiers

Le scan est lancé depuis le sous-dossier `app/` contenant le `pom.xml`. Maven doit d'abord être exécuté pour peupler le cache local `~/.m2` et éviter les erreurs de rate-limiting de Maven Central.

```powershell
cd app
mvn dependency:resolve
cd ..
trivy fs . --format json --output trivy-report.json
```

![Scan Trivy en cours](images/exo3/Capture%20d'écran%202026-06-18%20132511.png)

---

## 4. Identifier les dépendances vulnérables

### Nombre total de vulnérabilités

```powershell
(Get-Content trivy-report.json | ConvertFrom-Json).Results | ForEach-Object { $_.Vulnerabilities } | Measure-Object | Select-Object Count
```

![Total 126 vulnérabilités](images/exo3/Capture%20d'écran%202026-06-18%20132647.png)

**Résultat : 126 vulnérabilités détectées.**

### Répartition par sévérité

```powershell
(Get-Content trivy-report.json | ConvertFrom-Json).Results | ForEach-Object { $_.Vulnerabilities } | Group-Object Severity | Sort-Object Count -Descending | Select-Object Count, Name
```

![Répartition par sévérité](images/exo3/Capture%20d'écran%202026-06-18%20132708.png)

| Sévérité | Nombre |
| --- | --- |
| HIGH | 56 |
| MEDIUM | 47 |
| CRITICAL | 15 |
| LOW | 8 |

---

## 5. CVE les plus critiques

```powershell
(Get-Content trivy-report.json | ConvertFrom-Json).Results | ForEach-Object { $_.Vulnerabilities } | Where-Object { $_.Severity -eq "CRITICAL" } | Select-Object VulnerabilityID, PkgName, InstalledVersion, FixedVersion | Format-Table
```

![Liste des CVE critiques](images/exo3/Capture%20d'écran%202026-06-18%20132724.png)

| CVE | Dépendance | Version installée | Version corrigée |
| --- | --- | --- | --- |
| CVE-2019-17571 | log4j:log4j | 1.2.17 | — |
| CVE-2022-23305 | log4j:log4j | 1.2.17 | — |
| CVE-2022-23307 | log4j:log4j | 1.2.17 | — |
| CVE-2016-1000031 | commons-fileupload | 1.3.2 | 1.3.3 |
| CVE-2015-7501 | commons-collections4 | 4.0 | 4.1 |
| CVE-2022-47937 | org.apache.sling.commons.json | 2.0.4-incubator | — |
| CVE-2025-24813 | tomcat-embed-core | 9.0.36 | 11.0.3 / 10.1.35 / 9.0.99 |
| CVE-2026-41293 | tomcat-embed-core | 9.0.36 | 11.0.3 / 10.1.35 / 9.0.99 |
| CVE-2026-43512 | tomcat-embed-core | 9.0.36 | 11.0.3 / 10.1.35 / 9.0.99 |
| CVE-2026-43515 | tomcat-embed-core | 9.0.36 | 11.0.3 / 10.1.35 / 9.0.99 |
| CVE-2022-22965 | spring-boot-starter-web | 2.3.1.RELEASE | 2.5.12 / 2.6.6 |
| CVE-2022-22965 | spring-beans | 5.2.7.RELEASE | 5.2.20.RELEASE / 5.3.18 |
| CVE-2016-1000027 | spring-web | 5.2.7.RELEASE | 6.0.0 |
| CVE-2017-1000487 | plexus-utils | 1.0.4 | 3.0.16 |

---

## 6. Plan de mise à jour

Les vulnérabilités les plus critiques concernent **Log4j 1.x**, **Apache Tomcat** et **Spring Framework**. Voici les actions prioritaires :

### Priorité 1 – Log4j (3 CVE critiques, pas de fix sur la branche 1.x)

Log4j 1.2.17 est en fin de vie depuis 2015 et ne sera plus corrigé. Les CVE détectées permettent l'exécution de code arbitraire à distance (RCE).

**Action :** Migrer vers **Log4j 2.x** (≥ 2.17.1) ou remplacer par SLF4J + Logback.

### Priorité 2 – Apache Tomcat (4 CVE critiques)

La version 9.0.36 embarquée contient plusieurs vulnérabilités critiques.

**Action :** Mettre à jour `spring-boot-starter-parent` vers une version récente qui embarque Tomcat ≥ 9.0.99.

### Priorité 3 – Spring Framework (CVE-2022-22965 – Spring4Shell)

Spring4Shell est une vulnérabilité RCE très connue sur Spring 5.2.x et 5.3.x avant les versions correctives.

**Action :** Mettre à jour vers Spring Boot ≥ 2.6.6 ou ≥ 2.5.12.

### Priorité 4 – Commons FileUpload et Collections

**Action :** Passer `commons-fileupload` en 1.3.3 et `commons-collections4` en 4.1.

---

### Combien de vulnérabilités sont détectées ?

**126 vulnérabilités** ont été détectées au total : 15 critiques, 56 élevées, 47 moyennes et 8 faibles.

### Quelle est la vulnérabilité la plus critique ?

Les trois CVE sur **Log4j 1.2.17** (CVE-2019-17571, CVE-2022-23305, CVE-2022-23307) sont parmi les plus dangereuses car elles permettent une exécution de code arbitraire à distance et aucun correctif n'existe sur cette branche — la seule solution est la migration vers Log4j 2.x. **CVE-2022-22965 (Spring4Shell)** est également à considérer comme critique : c'est une RCE sur Spring Framework largement documentée et exploitée.

### Quelle dépendance est concernée ?

Les dépendances les plus touchées sont **log4j:log4j 1.2.17** (3 CVE critiques, fin de vie), **org.apache.tomcat.embed:tomcat-embed-core 9.0.36** (4 CVE critiques) et **org.springframework:spring-beans 5.2.7.RELEASE** (Spring4Shell).

### Quelle version corrige le problème ?

Pour Log4j, il n'existe pas de version 1.x corrective — il faut migrer vers Log4j 2 ≥ 2.17.1. Pour Tomcat, la version 9.0.99 corrige les CVE identifiées. Pour Spring4Shell, les versions 5.2.20.RELEASE ou 5.3.18 apportent le correctif, ou une migration vers Spring Boot ≥ 2.6.6.

