# Job Application Tracker (Gmail → Gemini → Google Sheets → Telegram)

Automatisation n8n qui suit les candidatures à partir de la boîte Gmail. Chaque email de recrutement est détecté, classé par Gemini, inscrit dans un Google Sheet, puis notifié sur Telegram. Le bot Telegram permet aussi d'agir sur les refus et de poser des questions sur le suivi.

Profil recherché : Full Stack Engineer, AI Engineer, Graduate Programs.

- Spécification fonctionnelle : https://claude.ai/artifact/LAhDKWrU1MQb25De8opXbU
- Instance actuelle : n8n Cloud (https://bagus-full-stack.app.n8n.cloud)
- Migration prévue : n8n self-hosted (voir la section dédiée)

## Sommaire

1. [Fonctionnement](#fonctionnement)
2. [Stack](#stack)
3. [Workflows](#workflows)
4. [Modèle de données](#modèle-de-données)
5. [Utilisation au quotidien](#utilisation-au-quotidien)
6. [Installation sur une instance neuve](#installation-sur-une-instance-neuve)
7. [Migration vers le self-hosting](#migration-vers-le-self-hosting)
8. [Sécurité et RGPD](#sécurité-et-rgpd)
9. [État du projet et limites connues](#état-du-projet-et-limites-connues)

## Fonctionnement

```
Gmail Trigger (polling 1 min)
  └─ gmail-watcher : filtre recrutement
       ├─ email-classifier : Gemini (via gemini-chain) → POSITIF / REFUS / EN_ATTENTE / HORS_SCOPE
       └─ update-tracker : upsert dans Sheets + notification Telegram
                              └─ REFUS : boutons [Feedback] [Archiver] [Ignorer]

Telegram (un seul bot, un seul webhook)
  └─ action-refus
       ├─ bouton  → send-feedback-request / archivage / ignorer / archivage par lot
       └─ texte   → chat-telegram : question sur le tableau ou archivage des refus d'une date

Planifiés : relance-tracker (tous les jours 9h), weekly-summary (lundi 8h)
Erreurs   : error-handler (alerte Telegram + ligne dans l'onglet logs)
```

Règles de conception :
- Aucun email n'est envoyé ni supprimé automatiquement. Le feedback est un **brouillon Gmail** relu à la main, l'archivage retire seulement le label `INBOX`.
- Le texte des emails et du Sheet est transmis à Gemini dans des blocs `<email>` / `<tableau>` avec la consigne de les traiter comme des données, jamais comme des instructions.
- Gemini ne décide que de l'intention et de la classification. Les comptages et les sélections de lignes à archiver sont faits par des nœuds déterministes.

## Stack

| Rôle | Outil |
|---|---|
| Orchestration | n8n (Cloud aujourd'hui, self-hosted ensuite) |
| Source | Gmail, Gmail Trigger natif en polling (1 minute), OAuth2 |
| IA | Google Gemini API, chaîne de 5 modèles (free tier) |
| Base de données | Google Sheets |
| Notifications et commandes | Telegram Bot API |

Chaîne de repli, définie une seule fois dans le sous-workflow `gemini-chain` (appelé par `email-classifier`, `chat-telegram`, `send-feedback-request` et `weekly-summary`, qui lui passent `system`, `user`, `json`, `maxTokens` et `temperature`) : les deux premiers modèles sont essayés une seule fois (une erreur 429 de quota journalier ne se résout pas en 5 s), les trois suivants 3 fois à 5 s d'intervalle. Le modèle suivant prend la main à chaque échec.

1. `gemini-3.5-flash`
2. `gemini-3.6-flash`
3. `gemini-3.5-flash-lite` (sans `thinkingBudget` : il renvoie 400 avec `thinkingBudget: 0`)
4. `gemini-3.1-flash-lite`
5. `gemini-3-flash-preview`

Écartés après mesure : `gemini-2.5-flash` et `gemini-2.5-flash-lite` (404 avec cette clé), `gemini-3.7-flash` et `gemini-3.8-flash` (503, ou plus de 150 s de réponse jusqu'à 300 s sans réponse).

Le free tier limite à environ 20 requêtes par jour et par modèle. Quand les deux premiers modèles sont épuisés (429), chaque appel perd quelques secondes avant d'atteindre un modèle qui répond. Si les 5 échouent, `email-classifier` renvoie `EN_ATTENTE` + `review_manuel` ; les trois autres workflows échouent et `error-handler` alerte. Les nœuds Gemini n'ont pas de limite de temps propre.

## Workflows

| Workflow | Déclencheur | Rôle |
|---|---|---|
| `gmail-watcher` | Gmail Trigger, 1 min | Filtre les emails de recrutement, appelle `email-classifier` puis `update-tracker`. Error workflow : `error-handler`. |
| `email-classifier` | Appelé | Gemini classe et extrait (entreprise, poste, contact, prochaine étape, langue, confiance, priorité 1 à 10). Un Code node valide la sortie, avec repli `EN_ATTENTE` + `review_manuel`. Appelle `gemini-chain` (voir Stack). |
| `gemini-chain` | Appelé | Essaie les 5 modèles Gemini dans l'ordre et renvoie la réponse du premier qui répond. Si les 5 échouent, il échoue : l'appelant décide (le classifier replie sur `EN_ATTENTE`). |
| `update-tracker` | Appelé | `HORS_SCOPE` : ligne dans `logs` uniquement. Sinon upsert sur `Email ID` dans `candidatures`, puis message Telegram. Les REFUS ont trois boutons. |
| `action-refus` | Telegram Trigger (boutons et texte) | Route les boutons par préfixe (`fb:`, `ar:`, `ig:`, `bk:`, `bn:`) et envoie le texte libre à `chat-telegram`. Error workflow : `error-handler`. |
| `send-feedback-request` | Appelé | Gemini rédige une demande de feedback FR/EN, Gmail crée un **brouillon** dans le fil d'origine, `Action` = `BROUILLON_FEEDBACK`. |
| `chat-telegram` | Appelé | Répond aux questions sur le Sheet (200 lignes les plus récentes) ou prépare l'archivage des refus d'une date, avec boutons de confirmation. |
| `archive-refus-date` | Appelé | Après confirmation : retire `INBOX` des refus de la date (50 maximum), `Action` = `ARCHIVÉ`, résumé Telegram. |
| `relance-tracker` | Tous les jours, 9h (Europe/Paris) | Rappel Telegram pour les `EN_ATTENTE` avec une entreprise renseignée, sans réponse depuis plus de 7 jours, sauf si une ligne `REFUS` ou `POSITIF` plus récente existe pour la même entreprise (et le même poste quand les deux sont renseignés). |
| `reclassify-review` | Manuel | Relit au maximum 10 lignes `EN_ATTENTE` dont la `Confiance IA` vaut 0 (classement Gemini échoué), puis relance `email-classifier` et `update-tracker` sur ces mails. Un mail reclassé `HORS_SCOPE` reçoit `Action` = `IGNORÉ` et n'est plus repris. À lancer après un jour de quota épuisé. |
| `weekly-summary` | Lundi, 8h (Europe/Paris) | Bilan de la semaine rédigé par Gemini. |
| `import-historique` | Manuel | Importe les 15 emails de recrutement les plus récents des 30 derniers jours (la requête Gmail reprend les mêmes termes que le filtre du watcher, avec une correspondance Gmail plus large que la regex ; le filtre n8n repasse derrière. Limite 15 pour rester sous le quota Gemini) avec le même filtre. `Action` et `Relance` = `IMPORT`, aucune notification. |
| `error-handler` | Error Trigger | Alerte Telegram (workflow, nœud, message, lien) + ligne `ERREUR` dans `logs`. |

Filtre de `gmail-watcher` (large à dessein : Gemini écarte les faux positifs en `HORS_SCOPE`). L'email passe si l'une de ces conditions est vraie :
- expéditeur : `recrutement|recruit|\brh@|\bhr@|talent|hiring|careers|jobs@` (`noreply@` retiré : il laissait passer presque tous les mails automatiques)
- sujet : `candidature|entretien|recrutement|offre d'emploi|poste d[e']|(le|ce|votre) poste|prochaine étape|retenu|(your|votre|an|the) application|application (for|to|received|status)|interview|recruit|job offer|(the|this|a|your) position|position (of|at|for)|next steps|assessment|test technique|technical test`
- corps : `nous avons le regret|non retenu|suite à votre candidature|unfortunately|not moving forward|we regret`

Un second nœud, `Not Job Alerts`, écarte ensuite les expéditeurs `jobalerts-noreply@`, `newsletters-noreply@` et `jobs-listings@` (alertes et newsletters LinkedIn, qui passaient par le mot `poste` du sujet et coûtaient une requête Gemini).

## Modèle de données

Google Sheet « Job Application Tracker », deux onglets.

**`candidatures`**

| Colonne | Contenu |
|---|---|
| Date | Date de réception (texte ISO, ex. `2026-10-25T10:00:00Z`) |
| Entreprise, Poste | Extraits par Gemini |
| Statut | `POSITIF` / `REFUS` / `EN_ATTENTE` (`HORS_SCOPE` n'est pas stocké) |
| Contact RH, Prochaine étape | Extraits par Gemini |
| Email ID | ID Gmail, clé de déduplication |
| Action | `—` / `ARCHIVÉ` / `IGNORÉ` / `BROUILLON_FEEDBACK` / `IMPORT` |
| Confiance IA | 0.0 à 1.0 |
| Langue | `FR` / `EN` |
| Priorité | 1 à 10, attribuée par Gemini |
| Relance | `NON` / `OUI` / `IMPORT` (`IMPORT` = jamais relancé) |

**`logs`** : Timestamp, Email ID, Workflow, Statut exécution (`SUCCESS` / `ERREUR`), Message.

Corriger un statut à la main dans le Sheet est toujours possible.

## Utilisation au quotidien

- Un email de recrutement arrive : une notification Telegram part en moins de 2 minutes (🟢 positif, 🔴 refus, 🟡 en attente).
- Sur un refus : **Feedback** crée un brouillon à relire dans Gmail, **Archiver** retire le mail de la boîte de réception, **Ignorer** ne fait rien.
- Écrire au bot en texte libre :
  - `Combien de refus ai-je ?`, `Quelles candidatures ont une priorité 10 ?` : réponse en lecture seule.
  - `archive tous les refus d'hier` : le bot annonce « Archiver N refus du AAAA-MM-JJ ? », rien ne se passe avant le clic sur ✅.
- Toute autre demande (envoyer, supprimer, modifier une ligne) est refusée : le bot ne sait que lire et archiver les refus d'une date.

## Installation sur une instance neuve

1. Créer les credentials n8n : Gmail OAuth2, Google Sheets OAuth2, Google Gemini (clé API depuis Google AI Studio), Telegram (token du bot créé avec @BotFather).
2. Créer le Google Sheet avec les deux onglets et les colonnes ci-dessus.
3. Récupérer l'ID de chat Telegram (ex. via @userinfobot) : il est écrit dans les nœuds Telegram.
4. Importer les workflows dans cet ordre, car un workflow publié ne peut référencer que des sous-workflows publiés : `error-handler`, `gemini-chain`, `email-classifier`, `update-tracker`, `send-feedback-request`, `archive-refus-date`, `chat-telegram`, `action-refus`, `relance-tracker`, `weekly-summary`, `gmail-watcher`. Garder `import-historique` non publié, il se lance à la main.
5. Remplacer l'ID du Sheet (18 occurrences dans `workflows\`) et l'ID de chat (9 occurrences) **avant l'import**, avec ce PowerShell lancé à la racine du projet (testé sur une copie : 0 ancien ID restant, JSON valides) :
   ```powershell
   $old = '16b0IWlJ9RBPFeaQ863YyTHmm4pWztvT-SxerLX5Q5Fw'; $new = 'NOUVEL_ID_SHEET'   # puis 911560431 -> NOUVEL_ID_CHAT
   Get-ChildItem workflows\*.json | ForEach-Object {
     $t = [IO.File]::ReadAllText($_.FullName).Replace($old, $new)
     [IO.File]::WriteAllText($_.FullName, $t, (New-Object Text.UTF8Encoding $false))
   }
   ```
   Ensuite, rebrancher les nœuds Execute Workflow sur les nouveaux IDs, définir `error-handler` comme error workflow de `gmail-watcher`, `action-refus`, `relance-tracker` et `weekly-summary`.
6. Publier, puis tester avec des emails réels (viser plus de 95 % de classification correcte sur 20 emails).

L'error workflow ne se déclenche que sur les exécutions de production, pas sur les tests manuels.

## Migration vers le self-hosting

### Pourquoi et ce que ça change

n8n Cloud est limité à un essai. Le self-hosting supprime cette limite et garde les données sur ta machine, mais il te donne en échange l'exploitation : mises à jour, sauvegardes, certificats. Il ne change rien au RGPD côté Gemini : les emails partent toujours chez Google.

Mode recommandé : **single** (un seul processus n8n, SQLite) derrière **Caddy** pour le HTTPS automatique. Le mode queue (Postgres, Redis, workers) ne se justifie pas pour un seul utilisateur.

### Prérequis

- Un VPS Ubuntu ou Debian, 2 Go de RAM minimum, accès SSH avec clé.
- Un nom de domaine (ex. `n8n.tondomaine.com`) avec un enregistrement DNS **A** vers l'IP du VPS. Un HTTPS public est obligatoire : Telegram n'envoie ses webhooks qu'à une URL HTTPS valide.
- Ports 80 et 443 ouverts, y compris dans le firewall du fournisseur.

### Étape 1 : préparer le serveur

```bash
# le DNS doit déjà pointer sur le VPS
dig +short n8n.tondomaine.com
curl -s ifconfig.me          # les deux IP doivent être identiques

# Docker + plugin Compose
curl -fsSL https://get.docker.com | sh
docker compose version

# firewall : seulement SSH, HTTP, HTTPS
ufw allow OpenSSH && ufw allow 80 && ufw allow 443 && ufw enable
```

Ne jamais ouvrir le port 5678 de n8n : seul Caddy est exposé.

### Étape 2 : déployer n8n

Dans `/opt/n8n`, placer un `docker-compose.yml` (services `n8n` et `caddy`), un `caddy_config/Caddyfile` qui fait le reverse proxy vers `n8n:5678`, et un `.env` (droits 600) :

```
DATA_FOLDER=/opt/n8n
DOMAIN_NAME=tondomaine.com
SUBDOMAIN=n8n
SSL_EMAIL=toi@exemple.com
GENERIC_TIMEZONE=Europe/Paris
N8N_ENCRYPTION_KEY=<généré sur le serveur>
```

Générer la clé **sur le serveur** et ne pas la réutiliser depuis une autre instance :

```bash
openssl rand -hex 32
```

La clé `N8N_ENCRYPTION_KEY` chiffre tous les credentials. Si elle est perdue ou modifiée, ils deviennent illisibles : la sauvegarder hors du VPS (gestionnaire de mots de passe).

Épingler la version de l'image n8n (pas de `latest`), puis :

```bash
cd /opt/n8n && docker compose up -d
docker compose ps
curl -fsS --retry 5 --retry-delay 10 https://n8n.tondomaine.com/healthz   # {"status":"ok"}
```

Premier certificat : compter une à deux minutes. Ouvrir ensuite `https://n8n.tondomaine.com` et **créer le compte owner immédiatement** (le premier arrivé prend l'instance), puis activer la 2FA.

### Étape 3 : exporter depuis n8n Cloud

Sur n8n Cloud, il n'y a pas de CLI : exporter chaque workflow depuis l'éditeur (menu `…` > Download) pour obtenir un fichier JSON. Les 11 workflows sont à exporter.

Les credentials ne s'exportent pas avec leurs secrets : ils sont à recréer sur la nouvelle instance.

### Étape 4 : recréer les credentials

| Credential | Ce qu'il faut faire |
|---|---|
| Gmail et Google Sheets | Créer un projet Google Cloud, activer les API Gmail et Sheets, configurer l'écran de consentement, créer un client OAuth « Application Web ». URI de redirection autorisée : `https://n8n.tondomaine.com/rest/oauth2-credential/callback`. Les saisir dans n8n puis autoriser le compte. Le OAuth « en un clic » de n8n Cloud n'existe pas en self-host. |
| Gemini | Saisir la clé API (Google AI Studio). |
| Telegram | Saisir le token du bot. Le bot existant peut être gardé, mais voir l'étape 6. |

### Étape 5 : importer et rebrancher les workflows

1. Importer les JSON (menu `…` > Import from file) dans l'ordre de la section « Installation ».
2. Rattacher les credentials recréés aux nœuds concernés.
3. Les IDs de workflows changent : rebrancher chaque nœud **Execute Workflow** sur le bon sous-workflow, et redéfinir `error-handler` comme error workflow.
4. Le Google Sheet ne change pas : les workflows pointent sur le même ID, l'historique est conservé.
5. Publier dans l'ordre : sous-workflows d'abord, `gmail-watcher` en dernier.

Si le CLI est utilisé plus tard entre deux instances self-hosted : `n8n export:workflow --all --output=...` puis `n8n import:workflow --input=...`.

### Étape 6 : basculer sans doublons

Un bot Telegram n'a qu'un seul webhook et un email ne doit pas être traité deux fois. Ordre à respecter :

1. Sur n8n Cloud, dépublier `gmail-watcher` et `action-refus`.
2. Sur le VPS, publier `action-refus` (le webhook Telegram se réenregistre sur la nouvelle URL), puis `gmail-watcher`.
3. Faire un test réel : envoyer un email de test à la boîte, vérifier la notification et la ligne dans le Sheet, puis écrire une question au bot.
4. Laisser les autres workflows planifiés (`relance-tracker`, `weekly-summary`) dépubliés sur le Cloud pour ne pas recevoir deux rappels.

Si `gmail-watcher` reste actif sur les deux instances, les mails sont traités en double, mais l'upsert sur `Email ID` évite les doublons de lignes. Seules les notifications seraient doublées.

### Étape 7 : exploitation

- **Sauvegardes** : le dossier de données (`/opt/n8n`, volume n8n contenant la base SQLite) et la clé de chiffrement, régulièrement et hors du serveur.
- **Mises à jour** : changer volontairement le tag de l'image, relire les notes de version, `docker compose pull && docker compose up -d`. Faire une sauvegarde avant.
- **Supervision** : `error-handler` continue de t'alerter sur Telegram. Surveiller aussi l'espace disque et l'expiration des certificats (Caddy les renouvelle seul).
- **Changer un réglage** : modifier `.env` puis `docker compose up -d`, pas de variable en dur dans le compose.

### Retour arrière

Tant que les workflows du Cloud existent, remettre `gmail-watcher` et `action-refus` en publié sur le Cloud et les dépublier sur le VPS suffit à revenir en arrière. Le Sheet est partagé, rien n'est perdu.

## Sécurité et RGPD

- Secrets uniquement dans les credentials n8n et dans `.env` (jamais dans les workflows ni dans git).
- Corps d'email tronqués à 500 caractères avant envoi à Gemini.
- Seules données stockées : le Google Sheet. Partage avec des tiers : Google Gemini et Telegram.
- **À arbitrer** : sur le free tier de Gemini, Google peut utiliser les contenus envoyés pour améliorer ses produits. L'offre payante n'entraîne pas les modèles sur les données. Les emails de candidature contiennent des données personnelles : décider avant d'aller en production.
- Scope Gmail : lecture et modification de labels, aucune suppression ni envoi.

## État du projet et limites connues

Fait et testé en simulation : détection, classification, mise à jour du Sheet, notifications, boutons Ignorer et alertes d'erreur, résumé hebdomadaire, chat Telegram en lecture, demande de confirmation d'archivage.

Pas encore validé en conditions réelles :
- l'archivage Gmail d'un vrai mail (seul un identifiant factice a été testé) ;
- le brouillon de feedback (qualité, destinataire, fil de discussion) ;
- le premier rappel `relance-tracker` ;
- le résumé hebdomadaire sans aucun positif ;
- le taux de 95 % de classification sur environ 20 emails réels ;
- `import-historique` (lancé une fois, résultat à contrôler dans le Sheet).

Limites :
- `archive-refus-date` : les dates sont comparées au jour **UTC** du mail, et si aucun archivage ne réussit, aucun résumé n'est envoyé.
- Les messages Telegram sont envoyés en Markdown : les champs dynamiques (entreprise, poste, contact, message d'erreur) sont échappés (`_ * ` [`). Tout nouveau message Telegram avec du texte variable doit faire pareil.
- `chat-telegram` ne transmet à Gemini que les 200 lignes les plus récentes.
- Le texte du Sheet part chez Gemini à chaque question.
- Le comportement de Gemini sur les sorties JSON et sur des emails inhabituels n'est pas garanti.

À faire : supprimer les lignes de test du Sheet (`test-upd-1`, `test-e2e-1`, `test-e2e-3`, `test-btn-1` et les lignes de test de `logs`), trancher la question RGPD, supprimer l'ancien bot Telegram inutilisé.
