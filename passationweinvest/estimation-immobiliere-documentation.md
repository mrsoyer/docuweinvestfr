# 🏠 Estimation Immobilière

## 🎯 Vue d'ensemble

Le système d'estimation immobilière de Weinvest s'appuie sur deux workflows n8n principaux :
1. **Répartition des codes postaux par agences** : Attribution automatique des leads aux conseillers/agences selon la zone géographique
2. **Traitement des demandes d'estimation** : Gestion complète du lead depuis le formulaire jusqu'à la distribution aux conseillers

### 📊 Schéma global du système

```mermaid
graph TB
    subgraph "Workflow 1 - Répartition CP"
        A1[Schedule 1h] --> A2[TRUNCATE cp_mandataire]
        A2 --> A3[Get Agences/Users]
        A3 --> A4{Pour chaque<br/>conseiller}
        A4 --> A5[Calcul distances CP]
        A5 --> A6[INSERT cp_mandataire]
        A6 --> A4
    end
    
    subgraph "Workflow 2 - Estimation"
        B1[Webhook /newestim] --> B2[Decode Base64]
        B2 --> B3{Estimation<br/>existe?}
        B3 -->|Non| B4[CityScan API]
        B3 -->|Oui| B13[Return cache]
        B4 --> B5[Géocodage]
        B5 --> B6[Estimation prix]
        B6 --> B7[IA Mapping]
        B7 --> B8[Store estim]
        B8 --> B9[Find Conseiller]
        B9 --> B10[Create Netty]
        B10 --> B11[Send Notifs]
        B11 --> B12[Response]
    end
    
    subgraph "External Services"
        C1[CityScan API]
        C2[Anthropic Claude]
        C3[Netty CRM]
        C4[Gmail/Slack]
    end
    
    subgraph "Database"
        D1[(cp_mandataire)]
        D2[(estim)]
        D3[(users/agences)]
    end
    
    A6 -.->|Write| D1
    B9 -.->|Read| D1
    B8 -.->|Write| D2
    B3 -.->|Read| D2
    A3 -.->|Read| D3
    
    B4 --> C1
    B7 --> C2
    B10 --> C3
    B11 --> C4
    
    style A1 fill:#e1f5fe
    style B1 fill:#e1f5fe
    style C1 fill:#fff3e0
    style C2 fill:#fff3e0
    style C3 fill:#fff3e0
    style C4 fill:#fff3e0
```

## 🔄 Workflow 1 : Répartition des codes postaux par agences

### 📊 Description
- **URL** : https://sync.weinvest.app/workflow/m6pNeIhMMhqnrpms
- **Déclencheur** : Schedule Trigger (toutes les heures)
- **Objectif** : Créer et maintenir une table de correspondance entre codes postaux et conseillers/agences

### ⚙️ Étapes du processus

1. **Réinitialisation** : `TRUNCATE TABLE cp_mandataire`

2. **Récupération des données**
   ```sql
   SELECT * FROM (SELECT 
       COALESCE(u.user_id, a.id) AS id,
       -- Données consolidées agences/users
       -- Filtrage des utilisateurs internes
       -- Exclusion agence id=10 et user_id=170
   ) WHERE actif = true 
   AND (google_my_business != '' OR google_my_business IS NOT NULL)
   ```

3. **Analyse par conseiller/agence**
   - Pour chaque entité, récupération du user principal
   - Détermination du type : "agence" ou "conseiller"
   - Configuration selon la zone :
     - **Île-de-France** (75, 77, 78, 91, 92, 93, 94, 95) : distance 10km, limit 6
     - **Province** : distance 30km, limit 100

4. **Exécution du workflow de géolocalisation** : `hED86KLwdHs2CdaC`

### 📊 Schéma du processus de répartition

```mermaid
graph LR
    subgraph "Données sources"
        A1[Table users<br/>Conseillers actifs]
        A2[Table agences<br/>Agences actives]
    end
    
    subgraph "Filtrage"
        B1[Exclusions:<br/>- Emails internes<br/>- Agence ID 10<br/>- User ID 170]
        B2[Critères:<br/>- actif = true<br/>- google_my_business renseigné]
    end
    
    subgraph "Configuration zones"
        C1[IDF<br/>75,77,78,91,92,93,94,95<br/>Distance: 10km<br/>Limit: 6]
        C2[Province<br/>Autres départements<br/>Distance: 30km<br/>Limit: 100]
    end
    
    subgraph "Calcul distances"
        D1[Pour chaque CP France]
        D2[Calcul distance<br/>conseiller ↔ CP]
        D3[Tri par distance]
        D4[Application limite]
    end
    
    subgraph "Table cp_mandataire"
        E1[cp: code postal]
        E2[id_conseiller: user_id]
        E3[id_agence: agency_id]
        E4[distance: km]
    end
    
    A1 --> B1
    A2 --> B1
    B1 --> B2
    B2 --> C1
    B2 --> C2
    C1 --> D1
    C2 --> D1
    D1 --> D2
    D2 --> D3
    D3 --> D4
    D4 --> E1
    
    style C1 fill:#e3f2fd
    style C2 fill:#f3e5f5
```

## 🔄 Workflow 2 : Traitement des demandes d'estimation

### 📊 Description
- **URL** : https://sync.weinvest.app/workflow/9ASIe7OIsp9Oymrl
- **Déclencheur** : Webhook POST `/newestim`
- **Objectif** : Traiter les demandes d'estimation, enrichir les données et distribuer aux conseillers

### ⚙️ Étapes du processus

1. **Réception et préparation des données**
   - Décodage du champ `data` (base64 → JSON)
   - Ajout de la date actuelle
   - Extraction des informations du formulaire

2. **Vérification de l'unicité**
   - ID unique : `route + postalCode + city + email`
   - Vérification dans la table `estim`
   - Si existe déjà : retour de l'estimation existante

3. **Gestion newsletter** (si newsletter = 1)
   - Appel webhook : `https://sync-webhook.weinvest.app/webhook/newsletter`

4. **Géocodage et estimation CityScan**
   ```javascript
   // Authentification
   POST https://api.cityscan.fr/security/authenticate
   
   // Géocodage
   POST https://api.cityscan.fr/solution/geocode/address
   
   // Estimation
   POST https://api.cityscan.fr/solution/realty-valuation/resale
   ```

5. **Extraction et mapping des données avec IA**
   - Utilisation d'Anthropic Claude pour mapper les champs du formulaire
   - Conversion des valeurs numériques en descriptions textuelles
   - Mapping détaillé des types de biens, états, équipements, etc.

6. **Stockage de l'estimation**
   ```sql
   UPSERT INTO estim (id, estim) VALUES (...)
   ```

7. **Génération du message formaté**
   - Template structuré avec sections :
     - 👤 Informations du contact
     - 🏠 Informations du bien
     - 📍 Caractéristiques détaillées
     - 💰 Informations financières
     - 💸 Estimation (low, mid, high)

8. **Attribution au conseiller**
   ```sql
   SELECT *, users.email as useremail
   FROM cp_mandataire
   JOIN users ON cp_mandataire.id_conseiller = users.user_id
   WHERE cp = '[code_postal]'
   ORDER BY 
       CASE WHEN id_agence > 2 THEN 0 ELSE 1 END,
       distance ASC
   LIMIT 1
   ```

9. **Création dans Netty CRM**
   - Création du contact
   - Création de la tâche (type 4)
   - Liaison contact/conseiller/agence

10. **Notifications multiples**
    - Email au conseiller via Gmail
    - Notification Slack (#notif-fr-valuation_requests)
    - Sauvegarde Google Sheets (2 onglets : estimhzv2, estimv2)
    - Webhook CRM avec segments (84 et 91)

### 📊 Schéma du flux d'estimation

```mermaid
sequenceDiagram
    participant F as Formulaire Web
    participant W as Webhook n8n
    participant C as CityScan API
    participant AI as Claude AI
    participant DB as Database
    participant N as Netty CRM
    participant M as Notifications
    
    F->>W: POST /newestim (base64)
    W->>W: Decode base64
    W->>DB: Check estim exists
    
    alt Estimation existe
        DB-->>W: Return cached estim
        W-->>F: Response 200 (cached)
    else Nouvelle estimation
        W->>C: Authenticate
        C-->>W: Bearer token
        W->>C: Geocode address
        C-->>W: Coordinates
        W->>C: Get valuation
        C-->>W: Price estimation
        
        W->>AI: Map form fields
        AI-->>W: Structured data
        
        W->>DB: Store estimation
        W->>DB: Get conseiller (cp_mandataire)
        DB-->>W: Conseiller info
        
        W->>N: Create contact
        W->>N: Create task (type 4)
        N-->>W: Success
        
        par Notifications parallèles
            W->>M: Email conseiller
            W->>M: Slack notification
            W->>M: Google Sheets
            W->>M: CRM webhook
        end
        
        W-->>F: Response 200 (new)
    end
```

### 📊 Structure des données d'estimation

```mermaid
graph TD
    subgraph "Input Form Data"
        A1[Contact Info<br/>nom, prenom, email, tel]
        A2[Property Location<br/>route, postalCode, city]
        A3[Property Details<br/>type, area, rooms, floor]
        A4[Property Features<br/>elevator, parking, garden]
        A5[Property Condition<br/>état, DPE, bruit, luminosité]
    end
    
    subgraph "CityScan Response"
        B1[Geocoding<br/>lat, lng, formatted_address]
        B2[Valuation<br/>low, mid, high prices]
        B3[Market Data<br/>comparables, trends]
    end
    
    subgraph "AI Mapping"
        C1[Type mapping<br/>1→Maison, 2→Appart...]
        C2[État mapping<br/>1→Neuf, 2→Rafraîchi...]
        C3[Features extraction<br/>boolean→text description]
    end
    
    subgraph "Final Message"
        D1[👤 Contact Section]
        D2[🏠 Property Section]
        D3[📍 Details Section]
        D4[💰 Financial Section]
        D5[💸 Estimation Section]
    end
    
    A1 --> C1
    A2 --> B1
    A3 --> C2
    A4 --> C3
    A5 --> C3
    
    B1 --> D2
    B2 --> D5
    C1 --> D2
    C2 --> D3
    C3 --> D3
    
    style B1 fill:#e8f5e9
    style B2 fill:#e8f5e9
    style C1 fill:#fce4ec
    style C2 fill:#fce4ec
    style C3 fill:#fce4ec
```

## 🔗 Intégrations

### 📡 APIs utilisées
- **CityScan** : Géocodage et estimation immobilière
  - Authentification par Bearer token
  - Endpoints : `/geocode/address`, `/realty-valuation/resale`
- **Netty CRM** : Gestion contacts et tâches
  - API Key : `29843393-2425-450e-a481-c418b8a3877e`
- **Google Maps** : Enrichissement des données d'agences
- **Anthropic Claude** : IA pour mapping intelligent des champs

### 💾 Stockage des données
- **Table `cp_mandataire`** : Correspondance CP ↔ conseiller/agence
- **Table `estim`** : Cache des estimations
- **Google Sheets** : Archive des demandes (2 feuilles)

## 📈 Mapping des données

### 🏠 Types de biens
- 1 : Maison
- 2 : Appartement Simplex
- 3 : Appartement Duplex
- 4 : Appartement Triplex

### 🔧 États du bien
- 1 : Refait à neuf
- 2 : Rafraîchi
- 3 : Standard
- 4 : Rafraichissement nécessaire
- 5 : Travaux importants

### 📊 Autres mappings
- **Bruit** : 1 (Très bruyant) → 5 (Très calme)
- **Luminosité** : 1 (Sombre) → 5 (Très clair)
- **Exposition** : Nord, Sud, Est, Ouest et combinaisons
- **DPE** : A → G
- **Proximité** : 1 (Très éloignée) → 5 (Très proche)

## 🚨 Points d'attention

1. **Performance**
   - Workflow de répartition exécuté toutes les heures
   - Cache des estimations pour éviter les appels API redondants
   - Limite de 6 conseillers en IDF, 100 en province

2. **Gestion des erreurs**
   - Webhooks avec réponses différenciées selon le statut
   - Stockage des erreurs dans la table `estim`
   - `onError: continueRegularOutput` sur les étapes critiques

3. **Sécurité**
   - Authentification CityScan avec credentials sécurisés
   - Filtrage des emails internes Weinvest
   - ID unique pour éviter les doublons

4. **Priorités d'attribution**
   - Priorité aux agences avec id > 2
   - Tri par distance croissante
   - Un seul conseiller par demande

## 🔧 Configuration requise

### Variables d'environnement
- Credentials PostgreSQL
- API Key Netty
- Credentials CityScan
- OAuth Gmail
- OAuth Slack
- OAuth Google Sheets

### Tables PostgreSQL
```sql
-- Table cp_mandataire
CREATE TABLE cp_mandataire (
    cp VARCHAR,
    id_conseiller INTEGER,
    id_agence INTEGER,
    distance NUMERIC,
    -- autres champs...
);

-- Table estim
CREATE TABLE estim (
    id VARCHAR PRIMARY KEY,
    estim JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Webhooks
- `/newestim` : Réception des demandes d'estimation
- `/webhook/newsletter` : Inscription newsletter
- `/webhook/e670f46c-e9cd-43cf-9e98-b0174e6af83f` : CRM tracking 