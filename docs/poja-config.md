# Configuration Poja — Actions à faire dans la console

Récapitulatif des réglages à effectuer sur la **console Poja** pour que l'application fonctionne en préproduction. Sans ces actions, le déploiement réussit (CI/CD) mais l'application ne démarre pas correctement en runtime.

---

## 1. Variables d'environnement (env `preprod`)

Le `.env` local (`.gitignore`) n'existe **pas** sur AWS. Toutes les valeurs doivent être injectées dans la console Poja (section configuration de l'environnement).

| Variable | Valeur attendue | Utilisation |
|---|---|---|
| `DB_URL` | `jdbc:postgresql://...` (Neon) | Datasource JPA (`application.properties`) |
| `AWS_S3_BUCKET` | nom du bucket S3 Poja | `BucketComponent` (upload, download, presign) |
| `AWS_EVENTBRIDGE_BUS` | bus eventbridge Poja | `EventProducer` (events asynchrones) |
| `SENTRY_DSN` *(option)* | DSN Sentry | Monitoring des erreurs |

> **Source SES hardcodée** : `EmailConf` utilise `noreply@poja.io` (+ région `eu-west-3` fixée). Si on veut un autre expéditeur, modifier `EmailConf` **et** vérifier l'identité dans AWS SES.

---

## 2. Workers — Action requise (async email)

- Console Poja → environnement `preprod` → **Edit** (section Workers).
- Ajouter **au moins 1 worker** puis **Save**.
- Sans worker, les events SQS (`SendEmailRequested` → envoi des relevés de notes PDF) ne sont **jamais consommés**.
- Limite plan Basic : 2 workers ; Premium : 10.

> Une modification de la configuration Poja pousse sur la branche de l'environnement et redéploie.

---

## 3. SES — Identité vérifiée

- L'expéditeur actuel est `noreply@poja.io` (fourni par Poja).
- En **sandbox SES**, les destinataires doivent être des emails **vérifiés** → demander le **production access** dans AWS SES pour envoyer vers des adresses arbitraires (étudiants du seed).

---

## 4. Scheduled tasks (pas encore actives)

- Si envoi automatique/périodique des relevés : console Poja → menu **Scheduled Tasks**.
- Configurer : **Nom**, **Classe de l'event** (ex. `EmailUpdateTriggered`), **Expression cron**, **Event stack source** (worker concerné).

---

## 5. Déploiement

- Push sur `preprod`/`prod` déclenche `cd-compute.yml` (build SAM, Java 21 corretto, région `eu-west-3`).
- **Règle de formatage** : `./format.sh` doit être exécuté avant chaque commit, sinon le check CI échoue.
- Au **premier boot** de la Lambda :
  - `ddl-auto=update` crée/maj les tables (`app_user`, `cours`, `groupe`, …) ;
  - le seed (`SchoolDataSeeder`, CommandLineRunner idempotent) insère les données.
- Vérifier les **logs** dans la console Poja (paramètres/observabilité) après le premier déploiement.

---

## 6. Vérifications post-déploiement

1. `GET /ping` → `pong`.
2. Sur Neon : tables créées + comptes du seed (2 parcours, 37 cours, 4 groupes, 6 users, 24 examens, 7 affectations, 11 inscriptions).
3. Envoyer un email de test async (`GET /hello?to=...`) et vérifier la réception.
4. Tester upload/download S3 via `BucketComponent` si un endpoint l'utilise.

---

## Rappels bonnes pratiques Poja

- **Formater** avant tout push : `./format.sh` (Linux/macOS) ou `.\format.bat` (Windows).
- **Ne jamais modifier** un script de migration déjà exécuté (Flyway `V*__...`) — créer une nouvelle version.
- Privilégier les **Presigned URLs** (`BucketComponent.presign`) pour les téléchargements directs, afin de ne pas charger la Lambda.
- `application.properties` fait `spring.config.import=optional:file:.env` → sans effet sur Poja (fichier absent), les valeurs viennent des variables d'environnement de la console.