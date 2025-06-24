# 🏢 Flux Lead B2B

## 🎯 Vue d'ensemble

Le système de gestion des leads B2B de Weinvest comprend 4 workflows n8n (2 documentés actuellement) :
1. **Webhook Site Internet** : Réception des leads du formulaire "entreprendre" sur weinvest.fr
2. **Landing Pages Webflow** : Gestion des leads provenant des landing pages (mixte B2B/B2C)
3. [Workflow 3 - À documenter]
4. [Workflow 4 - À documenter]

## 📥 Sources de leads B2B

### 🔗 Canaux d'acquisition
- **Site principal weinvest.fr** : Formulaire "entreprendre"
- **Landing pages Webflow** :
  - `/votre-projet-immo` (avec option "Devenir agent immobilier")
  - `/offre-bailleur`
  - Autres landing pages configurées

### 📊 Types de projets B2B
- Devenir agent immobilier
- Offre bailleur
- Autres projets professionnels

## 🔄 Workflow 1 : Webhook Site Internet

### 📊 Description
- **URL** : https://sync.weinvest.app/workflow/r8aKCyik9NFfunUo
- **Endpoint** : POST `/b2b`
- **Authentification** : Basic Auth
- **Objectif** : Traiter les soumissions du formulaire "entreprendre" du site principal

### ⚙️ Étapes du processus

1. **Réception des données**
   - Webhook avec authentification basique
   - Réception des données du formulaire

2. **Traitement et enrichissement**
   ```javascript
   // Ajout des champs standards
   item.json.body.date = new Date().toLocaleString()
   item.json.body.firstName = item.json.body.prenom
   item.json.body.lastName = item.json.body.nom
   item.json.body.form = "entreprendre"
   item.json.body.origin = "website"
   
   // Décodage du champ data (base64 → JSON)
   if (item.json.body.data) {
     const dataString = Buffer.from(item.json.body.data, 'base64').toString()
     const dataObj = JSON.parse(dataString)
     // Intégration des propriétés décodées
   }
   ```

3. **Envoi vers Customer.io**
   ```json
   {
     "customer": {données_enrichies},
     "messageForm": "submit_form_entreprendre",
     "segment": 78,
     "eventValue": {données_enrichies}
   }
   ```

## 🔄 Workflow 2 : Landing Pages Webflow

### 📊 Description
- **URL** : https://sync.weinvest.app/workflow/P4H5U2pQb7AcIvpT
- **Déclencheur** : Webflow Trigger (site ID: 661e4810af50b7e4bc1e15b7)
- **Objectif** : Router et traiter les leads selon le type de landing page et projet

### ⚙️ Structure de routage

```
Webflow Trigger
├── If: publishedPath = "/votre-projet-immo"
│   ├── If2: Projet = "Devenir agent immobilier"
│   │   └── Traitement B2B spécifique
│   └── Autre projet
│       └── Traitement avec attribution conseiller
└── If1: publishedPath = "/offre-bailleur"
    └── Traitement offre bailleur
```

### ⚡ Branches de traitement

#### 1. Branch "Devenir agent immobilier" (B2B pur)
- Enrichissement des données (firstName, lastName, email, telephone)
- Marquage : `form = "entreprendre"`, `origin = "landing"`
- Envoi vers Customer.io (segment 78)
- Pas d'attribution de conseiller

#### 2. Branch "Votre projet immo" (Mixte B2B/B2C)
- Génération message avec Claude/OpenAI
- Attribution au conseiller selon code postal
- Création contact dans Netty
- Création tâche Netty (type 4)
- Notifications multiples :
  - Email au conseiller
  - Slack : `#notif-recrutement-ad-fr`
  - Google Sheets : `votre-projet-immo`
  - CRM webhooks (segments 91, 102)

#### 3. Branch "Offre bailleur"
- Process similaire à "Votre projet immo"
- Slack différent : `#notif-fr-leads-offre-bailleur`
- Google Sheets : `votre-projet-immo_HZ`

### 🔧 Attribution des conseillers

```sql
SELECT *, users.email as useremail
FROM cp_mandataire
JOIN users ON cp_mandataire.id_conseiller = users.user_id
WHERE cp = '[Code postal du lead]'
ORDER BY 
    CASE WHEN id_agence > 2 THEN 0 ELSE 1 END,
    distance ASC
LIMIT 1;
```

### 🤖 Génération de messages avec IA

Template de prompt :
```
📋 NOUVEAU LEAD WEINVEST - votre-projet-immo

👤 INFORMATIONS DU CONTACT :
- Civilité
- Nom
- Prénom
- Téléphone
- Email
- Code postal
- Votre projet
- Attribué à [nom_conseiller]
```

## 📊 Intégrations

### 🔗 Systèmes connectés

#### Customer.io
- **Endpoint** : `https://sync-webhook.weinvest.app/webhook/cioformb2b`
- **Segments** :
  - 78 : Leads B2B "entreprendre"
  - 91 : Submit form estimation
  - 102 : Votre projet

#### Netty CRM
- **API** : `https://webapi.netty.fr/apiv1/`
- **API Key** : `29843393-2425-450e-a481-c418b8a3877e`
- **Actions** :
  - Création de contacts
  - Création de tâches (type 4)
  - Attribution conseiller/agence

#### Notifications
- **Gmail** : `contact@weinvest.fr`
- **Slack** :
  - `#notif-recrutement-ad-fr` : Leads "Devenir agent"
  - `#notif-fr-leads-offre-bailleur` : Leads "Offre bailleur"

### 💾 Stockage des données

#### Google Sheets
- **Document** : `19f6jbJpCaEBDi5r0RlCAtwKHnLEaebLWQ7xJY-I6hDg`
- **Feuilles** :
  - `votre-projet-immo` (gid: 307878857)
  - `votre-projet-immo_HZ` (gid: 1238667055)

#### Champs stockés
- Informations contact (Nom, Prénom, Email, Téléphone)
- Code postal
- Type de projet
- Date de soumission
- Attribution conseiller (si applicable)

## 🚨 Points d'attention

### Gestion des emails
```javascript
// Vérification multiple des champs email
if (!data.email) {
  if (data['Email Contact']) {
    data.email = data['Email Contact'];
  } else if (data['Email newsletter']) {
    data.email = data['Email newsletter'];
  }
}
```

### Gestion des téléphones manquants
- Valeur par défaut : `"0600000000"` si téléphone = "null"

### Authentification
- Webhook B2B protégé par Basic Auth
- Credentials nécessaires pour toutes les intégrations

### Retry et gestion d'erreurs
- `retryOnFail: true` sur les appels Netty
- `waitBetweenTries: 5000` (5 secondes)
- `onError: continueRegularOutput` sur étapes non critiques

## 📈 Métriques et tracking

### Événements Customer.io
- `submit_form_entreprendre` : Formulaire site principal
- `d'information` : Landing pages
- `votre-projet` : Projets immobiliers

### Segmentation
- **Segment 78** : Leads B2B purs (entreprendre)
- **Segment 91** : Estimation
- **Segment 102** : Projet immobilier

### Points de tracking
1. Réception du lead (webhook)
2. Envoi Customer.io
3. Attribution conseiller (si applicable)
4. Création Netty
5. Notifications envoyées
6. Sauvegarde Google Sheets

## 🔧 Configuration requise

### Credentials nécessaires
- Basic Auth pour webhook B2B
- Webflow API
- Customer.io webhook auth
- Netty API Key
- Gmail OAuth
- Slack OAuth
- Google Sheets OAuth
- PostgreSQL (pour table cp_mandataire)

### IA Models
- Anthropic Claude (principal)
- OpenAI GPT (backup)

### Tables PostgreSQL
- `cp_mandataire` : Attribution codes postaux/conseillers
- `users` : Informations conseillers

## 🔄 Workflow 3 : Hub Customer.io & Segment

### 📊 Description
- **URL** : https://sync.weinvest.app/workflow/WTifS0HIrulm7o88
- **Endpoint** : POST `/cioformb2b`
- **Objectif** : Centraliser l'envoi des données vers Customer.io et Segment depuis les workflows 1 et 2

### ⚙️ Étapes du processus

1. **Génération ID unique**
   - Appel webhook : `https://sync-webhook.weinvest.app/webhook/76e1cd08-0527-4b5b-b7da-729829e500e7`
   - Paramètres : ID et email du customer
   - Attribution d'un ID Customer.io unique

2. **Enrichissement des données**
   ```javascript
   item.json.body.customer.date = new Date().toLocaleString()
   item.json.body.customer.messageForm = item.json.body.messageForm
   item.json.body.customer.data = ID_généré
   ```

3. **Envoi vers Segment**
   - **Identify** : Création/mise à jour du profil utilisateur
   - **Track Events** :
     - `formB2B` : Soumission formulaire B2B
     - `submit_form_entreprendre` : Formulaire entreprendre
     - `selected_newsletter_btb` : Inscription newsletter (si newsletter = 1)

4. **Envoi vers Customer.io**
   - Ajout au segment configuré (78, 91, 102...)
   - Déclenchement événements :
     - `thanks` : Accusé réception
     - `submit_form_entreprendre` : Event principal
     - `selected_newsletter_btb` : Si newsletter activée

### 📊 Configuration Segment
- **Write Key** : `EDcslpDGO6LLMoCwTBv3ueuxDM3BZnvP`
- **Events** : formB2B, submit_form_entreprendre, selected_newsletter_btb
- **Traits** : Toutes les données du formulaire

## 🔄 Workflow 4 : Scoring & Pipedrive

### 📊 Description
- **URL** : https://sync.weinvest.app/workflow/zJD3aFxlchU6g8MF
- **Endpoint** : POST `/b2b_send`
- **Déclencheur** : Appelé par Customer.io après traitement
- **Objectif** : Enrichir, scorer et créer/mettre à jour le lead dans Pipedrive

### ⚙️ Système de scoring

#### 📊 Sources de points
1. **Google Sheets - Points UTM** (gid: 785215617)
   - Points selon utm_source/utm_medium
   - Différenciation firstLead/OtherLead
   
2. **Google Sheets - Régions** : Attribution commerciale par région
   
3. **Google Sheets - Étiquettes** (gid: 222425881)
   - Mapping points → étiquette Pipedrive
   - Classification : chaud/tiède/froid

#### 🧮 Calcul des points
```javascript
// Points selon formulaire et statut
if(formB2B === 1) {
  points += formLigne.OtherLead  // Client existant
} else {
  points += formLigne.firstLead  // Nouveau client
}

// Points selon source/medium UTM
// Recherche hiérarchique : exact → source → medium
```

### ⚡ Flux de traitement

1. **Récupération données Customer.io**
   - API : `https://api-eu.customer.io/v1/customers/{user_id}/attributes`
   - Bearer Token : `32dcf26106c41e433b0f62e70400012d`

2. **Enrichissement et scoring**
   - Calcul points totaux (existants + nouveaux)
   - Attribution étiquette selon score
   - Attribution commercial selon région
   - Mapping statut → étiquette deal Pipedrive

3. **Recherche/Création dans Pipedrive**
   - Recherche par email
   - Si existe : UPDATE person
   - Si nouveau : CREATE person + lead

4. **Création du lead Pipedrive**
   - Titre : `Website - {nom}`
   - Attribution au commercial défini
   - Ajout notes avec détails MQL

5. **Notifications et tracking**
   - Slack : `#notif-fr-general-requests-b2b`
   - Google Sheets : Onglet B2B (gid: 850113397)
   - Segment : Event `scoringB2B` avec détails

### 🏷️ Mapping des étiquettes

#### Statuts professionnels → UUID Pipedrive
```javascript
{
  "Directeur d'agence": "b569cf80-3a06-11ef-a23b-812e2f1da651",
  "Agent commercial": "eb437490-355a-11ef-a5d6-397794110467",
  "Mandataire immobilier": "abe7b1e0-24ba-11ef-8be8-919108fddd20",
  "Négociateur immobilier": "2022e470-2796-11ef-b645-fb893d33c15d",
  "Reconversion professionnelle": "b21f2fc0-24ba-11ef-bf14-590256026f78",
  "Manager immobilier": "2022e470-2796-11ef-b645-fb893d33c15d",
  "Autre": "f028bb70-3ec7-11ef-80ba-3fb3cab51b5b"
}
```

### 📝 Notes et activités Pipedrive

1. **Note MQL** : Historique complet du scoring
2. **Note Message** : Si message facultatif fourni
3. **Note Landing** : Si source = "Contact Form Rent"
   - Présence RENT
   - Statut
   - Jeu concours
   - Entreprise

### 🚨 Gestion des cas spéciaux

1. **Conseil immobilier d'entreprise**
   - Attribution forcée : Maxime Zerbib (ID: 22567112)
   - Email : maxime.zerbib@weinvest.fr

2. **Délais de traitement**
   - Wait 1 minute avant récupération activités Customer.io
   - Permet la synchronisation des événements

3. **Champs personnalisés Pipedrive**
   - Mapping complet des champs UTM
   - Stockage ID Customer.io
   - Scoring et étiquettes

## 📊 Vue d'ensemble du flux complet B2B

```
1. Formulaire Web/Landing
   ↓
2. Workflow 1 ou 2 (selon source)
   ↓
3. Workflow 3 : Hub Customer.io/Segment
   ↓
4. Customer.io déclenche Workflow 4
   ↓
5. Scoring + Pipedrive + Notifications
```

## 🔧 Configuration complète requise

### Tables PostgreSQL
- `commonid` : Stockage des IDs uniques Customer.io
- `cp_mandataire` : Attribution géographique

### Google Sheets de configuration
1. **Points UTM** : Scoring par source marketing
2. **Régions** : Attribution commerciale
3. **Étiquettes** : Mapping score → classification
4. **B2B Archive** : Historique complet des leads

### APIs et credentials
- Customer.io API (Bearer token)
- Segment Write Key
- Pipedrive API (OAuth)
- Slack OAuth
- Google Sheets OAuth 