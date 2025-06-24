# 👥 Flux Lead B2C

## 🎯 Vue d'ensemble

Le système de gestion des leads B2C de Weinvest est centralisé dans un workflow n8n complexe qui gère :
- Les demandes de visite de biens immobiliers
- Les formulaires d'information et de contact
- L'attribution automatique aux conseillers/agences
- L'intégration avec Netty CRM et Pipedrive B2C
- Les notifications multi-canaux

## 📥 Sources de leads B2C

### 🔗 Canaux d'acquisition
- **Formulaire site principal** : Demandes de visite sur fiches biens
- **Formulaire contact** : Demandes d'information générales  
- **Pages agences** : Formulaires spécifiques par agence
- **Site pro** : Version pro du site (tracking séparé)

### 📊 Types de demandes
- Visite de bien spécifique (avec `bienid`)
- Contact général (sans bien)
- Contact via page agence (avec `agency`)
- Inscription newsletter

## 🔄 Workflow principal B2C

### 📊 Description
- **URL** : [Non fournie dans le JSON]
- **Endpoint** : POST `/b2c`
- **Déclencheur** : Webhook
- **Objectif** : Router et traiter tous les leads B2C selon leur type et origine

### ⚙️ Étapes du processus

1. **Réception et traitement initial**
   ```javascript
   // Décodage base64 du champ data
   const dataString = Buffer.from(item.json.body.data, 'base64').toString()
   const dataObj = JSON.parse(dataString)
   
   // Ajout des champs standards
   item.json.body.date = new Date().toLocaleString()
   item.json.body.firstName = item.json.body.prenom
   ```

2. **Routage selon le type de demande**
   - **If1** : Demande avec `bienid` → Recherche du bien dans Netty
   - **If3** : Demande avec `agency` → Attribution à l'agence
   - **If5** : Demande sans bien ni agence → Contact général
   - **If8** : Site pro vs standard → Archivage différencié

### 🏢 Branch 1 : Demande de visite avec bien

1. **Recherche du bien dans Netty**
   ```http
   GET https://webapi.netty.fr/apiv1/products
   ?filters=product_ref:equal:{bienid}
   ```

2. **Si bien trouvé** (count = 1) :
   - Création contact Netty avec user/company du bien
   - Création tâche type 17 (Demande de visite)
   - Liaison avec le bien (`linked_product_ref`)
   - Email au conseiller du bien
   - Envoi Customer.io segment 81

3. **Si bien introuvable** :
   - Archivage Google Sheets
   - Pas de création dans CRM

### 🏘️ Branch 2 : Contact via page agence

1. **Extraction ID agence**
   ```javascript
   agency.split("-")[agency.split("-").length-1]
   ```

2. **Attribution au conseiller principal**
   ```sql
   SELECT * FROM users 
   WHERE linked_company_id = {agency_id}
   ORDER BY user_profile DESC
   LIMIT 1
   ```

3. **Création dans Netty**
   - Contact lié au conseiller principal
   - Tâche type 17 avec description complète
   - Email au conseiller
   - Email à l'agence (table `agences`)

4. **Pipedrive B2C** (si agences 24, 26, 28) :
   - Création person avec tag "Marketing"
   - Création lead associé
   - Note avec détails complets

### 📝 Branch 3 : Contact général

Attribution selon présence `agency` dans le body :
- Si `agency` défini → même process que Branch 2
- Sinon → attribution par défaut ou archivage

### 📊 Intégrations

#### Netty CRM
- **API Key** : `29843393-2425-450e-a481-c418b8a3877e`
- **Endpoints** :
  - `/products` : Recherche de biens
  - `/contacts` : Création de contacts
  - `/tasks` : Création de tâches (type 17)

#### Pipedrive B2C
- **Credentials** : `b2c`
- **Owner ID** : 23046140
- **Actions** :
  - Création person avec custom field "Marketing"
  - Création lead "Demande de visite site we Invest"
  - Ajout notes formatées

#### Customer.io
- **Endpoint** : `https://sync-webhook.weinvest.app/webhook/e670f46c-e9cd-43cf-9e98-b0174e6af83f`
- **Segment** : 81
- **Event** : `submit_form_contact`

#### VisualQIE
- **URL** : `https://www.visualqie.com/getLeads.php`
- **Token** : `xkeyvqie-Sy43MXZ09HTUNTAwMTY0NjQwMDQ3tjS0NDMxsTQ3NzO1NDHR9a7M8PYJNvVycnS0hagzM0JWZ2JuamZhAAA=`
- **Envoi** : Leads avec `bienid`

### 💾 Stockage des données

#### Google Sheets
- **Document** : `19f6jbJpCaEBDi5r0RlCAtwKHnLEaebLWQ7xJY-I6hDg`
- **Feuilles** :
  - `B2C` (gid: 878641417) : Leads standard
  - `B2CPRO` (gid: 1753545984) : Leads site pro

#### Champs stockés
- Informations contact complètes
- Bien concerné (si applicable)
- Agence/conseiller attribué
- Date et origine
- Données décodées du formulaire

### 📧 Notifications

#### Emails (Gmail)
- **From** : `contact@weinvest.fr`
- **Templates** :
  - "Nouvelle demande de visite site we Invest"
  - "Nouveau lead issue de la page agence du site weInvest"
- **Destinataires** :
  - Conseiller du bien
  - Conseiller principal de l'agence
  - Email de l'agence

#### Slack
- **Channels** :
  - `#notif-recrutement-ad-fr` : Leads standard
  - `#notif-b2c-fr-pro` : Leads site pro
- **Format** : Markdown avec tous les détails

### 🚨 Points d'attention

1. **Gestion des biens inexistants**
   - Pas de création CRM si bien introuvable
   - Archivage uniquement en Google Sheets

2. **Attribution par défaut**
   - Si pas d'agence : recherche dans table `users`
   - Si agence invalide : utilisation du principal (user_profile DESC)

3. **Newsletter**
   - Endpoint séparé : `/webhook/newsletter`
   - Basic Auth requis
   - Traitement asynchrone

4. **Sites multiples**
   - Différenciation site standard vs pro
   - Archivage séparé
   - Canaux Slack différents

5. **Agences spéciales** (24, 26, 28)
   - Double intégration : Netty + Pipedrive
   - Owner Pipedrive fixe : 23046140

## 📈 Métriques et KPIs

### Points de tracking
1. Réception webhook initial
2. Type de demande identifié
3. Attribution conseiller/agence
4. Création CRM (Netty/Pipedrive)
5. Notifications envoyées
6. Archivage Google Sheets

### Segments Customer.io
- **81** : Tous les leads B2C

### Types de tâches Netty
- **17** : Demande de visite / Lead website

## 🔧 Configuration requise

### Tables PostgreSQL
- `users` : Conseillers et profils
- `agences` : Informations agences
- `properties` / `biens` : Catalogue immobilier

### Credentials
- Webhook auth (optionnel)
- Netty API Key
- Pipedrive B2C OAuth
- Gmail OAuth
- Slack OAuth
- Google Sheets OAuth
- Customer.io webhook auth
- VisualQIE token

### Variables spéciales
- `user_profile` : Ordre de priorité des conseillers
- `linked_company_id` : Liaison user → agence
- `product_ref` : Référence unique des biens

## 📊 Schéma de décision

```
Webhook B2C
├── If bienid exists
│   ├── Bien trouvé → Netty (contact + tâche) → Email conseiller
│   └── Bien non trouvé → Google Sheets uniquement
├── If agency exists
│   ├── User trouvé → Netty → Email user
│   └── User non trouvé → Agence principale → Netty → Email agence
└── Contact général
    └── Attribution par défaut ou archivage
``` 