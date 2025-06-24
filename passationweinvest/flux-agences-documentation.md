# 🏢 Automatisation - Flux des Agences et Utilisateurs

## 🏢 Flux Agences

### 🎯 Vue d'ensemble
Le système de synchronisation des agences récupère les données depuis l'API Netty, les enrichit avec des informations Google My Business, et maintient la table `agences` à jour.

### 📊 Schéma global du système

```mermaid
graph TB
    subgraph "Sources de données"
        A1[API Netty<br/>Agences]
        A2[API Netty<br/>Users]
        A3[Google My Business<br/>via SerpAPI]
        A4[Table Properties<br/>Comptage biens]
    end
    
    subgraph "Workflows n8n"
        B1[Workflow Agences<br/>Schedule]
        B2[Workflow Users<br/>10 min]
    end
    
    subgraph "Script Python"
        C1[Fusion données<br/>Agences + Users]
        C2[Calcul stats<br/>Biens par agent]
        C3[Génération slugs<br/>URLs uniques]
        C4[Sync Bubble<br/>Directory]
    end
    
    subgraph "Base de données"
        D1[(Table agences)]
        D2[(Table users)]
        D3[(Table directories)]
    end
    
    subgraph "Destinations"
        E1[Bubble Directory<br/>Annuaire web]
        E2[Bubble Search<br/>Autocomplétion]
        E3[Imgix CDN<br/>Cache images]
    end
    
    A1 --> B1
    A3 --> B1
    B1 --> D1
    
    A2 --> B2
    B2 --> D2
    B2 --> E3
    
    D1 --> C1
    D2 --> C1
    A4 --> C2
    
    C1 --> C2
    C2 --> C3
    C3 --> C4
    C4 --> D3
    C4 --> E1
    C4 --> E2
    
    style B1 fill:#e1f5fe
    style B2 fill:#e1f5fe
    style C1 fill:#f3e5f5
```

## 📥 Workflow 1 - Synchronisation des Agences (n8n)

### ⚡ Description

{% hint style="info" %}
**URL du workflow** : https://sync.weinvest.app/workflow/hc9I2FGQGRI1xwUX
{% endhint %}

Ce workflow n8n synchronise les données des agences immobilières depuis l'API Netty vers la base de données.

### 🔄 Fonctionnement détaillé

1. **⏰ Déclenchement programmé**
   - Exécution planifiée régulière (Schedule Trigger)

2. **📡 Récupération des données Netty**
   - Appel API : `https://webapi.netty.fr/apiv1/companies`
   - Récupération par lots de 10 avec pagination
   - Authentification via header `x-netty-api-key`

3. **🔄 Traitement des données**
   - Transformation des réseaux sociaux (format `social_networks_[NomReseau]`)
   - Transformation des départements couverts en deux champs :
     - `departments_covered_code` : codes départements
     - `departments_covered_name` : noms départements

4. **🌐 Enrichissement Google My Business** (si URL GMB disponible)
   - Récupération de l'ID place Google
   - Appel à l'API SerpAPI pour obtenir :
     - Horaires d'ouverture (lundi à dimanche)
     - Informations complémentaires
   - Conversion des horaires au format 24h

5. **⭐ Calcul de la note moyenne**
   - Requête SQL : `SELECT AVG(rating) FROM reviews WHERE agency_id = {company_id}`
   - Ajout de la note moyenne aux données de l'agence

6. **💾 Sauvegarde en base de données**
   - Upsert dans la table `agences` (INSERT ou UPDATE selon l'existence)
   - Clé de matching : `id` (company_id de Netty)

7. **🔔 Notification webhook**
   - Appel webhook après insertion : `https://sync-webhook.weinvest.app/webhook/...`
   - Transmission des IDs pour traitement ultérieur

### 📊 Champs synchronisés

| Champ Netty | Champ DB | Description |
|-------------|----------|-------------|
| company_id | id | Identifiant unique de l'agence |
| name | nom | Nom de l'agence |
| manager_full_name | nom_gerant | Nom du gérant |
| address | adresse | Adresse |
| postal_code | code_postal | Code postal |
| city | ville | Ville |
| email | email | Email principal |
| legal_siret_number | siret | Numéro SIRET |
| guarantee_fund_amount | montant_fonds_garantie | Montant du fonds de garantie |
| deleted | supprime | Statut de suppression |
| time_modified | date_modification | Date de dernière modification |
| social_networks_* | social_networks_* | URLs des réseaux sociaux |
| (calculé) | note_moyenne | Note moyenne des avis Google |

### 🔗 APIs et intégrations

- **API Netty** : Source principale des données agences
- **Google Maps/SerpAPI** : Enrichissement avec horaires et informations GMB
- **PostgreSQL** : Base de données de destination
- **Webhook** : Notification pour traitements complémentaires

### 📊 Schéma du workflow agences

```mermaid
sequenceDiagram
    participant S as Schedule Trigger
    participant N as Netty API
    participant W as Workflow n8n
    participant G as Google/SerpAPI
    participant DB as PostgreSQL
    participant WH as Webhook
    
    S->>W: Déclenchement
    
    loop Pagination
        W->>N: GET /companies (limit=10)
        N-->>W: Companies data
        
        loop Pour chaque agence
            W->>W: Transform social networks
            W->>W: Transform departments
            
            alt Has GMB URL
                W->>G: Get Place ID
                G-->>W: Place info
                W->>G: Get business details
                G-->>W: Opening hours
                W->>W: Convert to 24h format
            end
            
            W->>DB: SELECT AVG(rating)
            DB-->>W: Note moyenne
            
            W->>DB: UPSERT agence
            DB-->>W: Success
            
            W->>WH: Notify completion
        end
    end
```

---

## 👥 Flux Utilisateurs

### 🎯 Vue d'ensemble
Le système de synchronisation des utilisateurs (agents immobiliers) récupère les données depuis l'API Netty et maintient la table `users` à jour.

## 📥 Workflow 2 - Synchronisation des Utilisateurs (n8n)

### ⚡ Description

{% hint style="info" %}
**URL du workflow** : https://sync.weinvest.app/workflow/6U8aAj0SBHnW6IMP
{% endhint %}

Ce workflow n8n synchronise les données des utilisateurs/agents toutes les 10 minutes.

### 🔄 Fonctionnement détaillé

1. **⏰ Déclenchement programmé**
   - Exécution toutes les 10 minutes
   - Récupération de la dernière date de modification en base

2. **📡 Récupération des données Netty**
   - Appel API : `https://webapi.netty.fr/apiv1/users`
   - Tri par date de modification décroissante
   - Pagination : 3 pages de 100 utilisateurs (0, 100, 200)
   - Authentification via header `x-netty-api-key`

3. **🔍 Filtrage des données**
   - Filtre sur les utilisateurs ayant une `time_modified`
   - Évite de traiter les utilisateurs non modifiés

4. **💾 Sauvegarde en base de données**
   - Upsert dans la table `users`
   - Clé de matching : `user_id`
   - Gestion spéciale pour certains IDs (186, 196, 204) : `user_profile = 4`

5. **🖼️ Gestion du cache d'images**
   - Format avatar : `users-{user_id}.jpg`
   - Purge du cache Imgix après mise à jour
   - API Imgix avec Bearer token pour invalidation

### 📊 Champs synchronisés

| Champ Netty | Champ DB | Description |
|-------------|----------|-------------|
| user_id | user_id | Identifiant unique |
| activated | activated | Statut d'activation |
| first_name | first_name | Prénom |
| last_name | last_name | Nom |
| email | email | Email |
| phone_mobile | phone_mobile | Téléphone mobile |
| job | job | Fonction |
| linked_company_id | linked_company_id | ID de l'agence liée |
| legal_status | legal_status | Statut légal |
| departments_covered | departments_covered | Départements couverts |
| cities_covered | cities_covered | Villes couvertes |
| presentation_short | presentation_courte | Présentation courte |
| time_created | time_created | Date de création |
| time_modified | time_modified | Date de modification |
| deleted | deleted | Statut de suppression |
| (généré) | avatar | Nom du fichier avatar |

### 🔗 APIs et intégrations

- **API Netty** : Source principale des données utilisateurs
- **PostgreSQL** : Base de données de destination
- **Imgix API** : Gestion du cache des avatars

### ⚡ Optimisations

- **Synchronisation incrémentale** : Récupération uniquement des modifications récentes
- **Traitement par lots** : Pagination pour éviter les timeouts
- **Cache d'images** : Purge automatique pour forcer le rafraîchissement des avatars

### 🚨 Points d'attention

- Les workflows agences et utilisateurs sont liés (linked_company_id)
- La fréquence de 10 minutes pour les users assure une synchronisation quasi temps réel
- Les IDs spéciaux (186, 196, 204) ont un traitement particulier pour le user_profile
- Le cache Imgix doit être purgé pour que les nouveaux avatars soient visibles

### 📊 Schéma du workflow utilisateurs

```mermaid
graph LR
    subgraph "Déclenchement"
        A1[Schedule<br/>10 min]
        A2[Get last<br/>time_modified]
    end
    
    subgraph "Récupération Netty"
        B1[Page 0<br/>0-99]
        B2[Page 1<br/>100-199]
        B3[Page 2<br/>200-299]
    end
    
    subgraph "Traitement"
        C1[Filter<br/>time_modified exists]
        C2[Special IDs<br/>186,196,204]
        C3[Set user_profile=4]
    end
    
    subgraph "Sauvegarde"
        D1[UPSERT users<br/>by user_id]
        D2[Generate avatar<br/>users-{id}.jpg]
    end
    
    subgraph "Cache"
        E1[Imgix API<br/>Purge cache]
        E2[Bearer token<br/>Authentication]
    end
    
    A1 --> A2
    A2 --> B1
    A2 --> B2
    A2 --> B3
    
    B1 --> C1
    B2 --> C1
    B3 --> C1
    
    C1 --> C2
    C2 -->|Yes| C3
    C2 -->|No| D1
    C3 --> D1
    
    D1 --> D2
    D2 --> E1
    E1 --> E2
    
    style A1 fill:#e1f5fe
    style C2 fill:#fff3e0
    style E1 fill:#f3e5f5
```

---

## 🔄 Workflow 3 - Alimentation Bubble et Table Directory (Python)

### 🎯 Vue d'ensemble
Ce script Python combine les données des agences et utilisateurs pour créer un annuaire unifié, calcule des statistiques sur les biens, et synchronise le tout avec Bubble et la table `directories`.

### 📝 Script de synchronisation

**Fichier** : Script Python de synchronisation des directories

**Utilisation** :
```bash
python sync_directories.py [options]

OPTIONS:
    --test    Utilise l'URL de test de l'API Bubble
```

### 🔄 Fonctionnement détaillé

1. **🔗 Fusion des données agences/utilisateurs**
   - Requête SQL complexe avec UNION ALL
   - Deux branches principales :
     - Agence ID 2 : Users considérés comme indépendants
     - Autres agences : Users rattachés à leur agence
   - Exclusion des emails internes Weinvest
   - Exclusion des agences ID 10 et 115

2. **📊 Calcul des statistiques**
   - **Comptage des biens actifs** par agent/agence :
     - Biens en vente (`type_offre = 1`)
     - Biens en location (`type_offre = 2`)
   - **Critères de comptage** :
     - `_id IS NOT NULL` (synchronisé avec Bubble)
     - `deleted = false`
     - `en_ligne = true`
     - `state = 1 OR state = 6`
   - **Order** : Nombre total de biens + 10000 pour les agences (pour priorisation)

3. **🏷️ Génération des slugs**
   - Pour les agents (agency_id = 2 ou annuaire = false) :
     - Format : `{nom-agent}-{user_id}`
   - Pour les agences :
     - Format : `{nom-agence}-{agency_id}`
   - Normalisation : suppression accents, espaces remplacés par tirets

4. **🔍 Gestion des termes de recherche**
   - Création d'un array `search` avec : nom, nom_agence, adresse, code_postal, ville
   - Vérification dans `searchdirectory` Bubble
   - Ajout des nouveaux termes pour l'autocomplétion

5. **🔄 Synchronisation Bubble**
   - **Conditions de synchronisation** : `actif = true`
   - **Création** : Si pas d'ID Bubble existant
   - **Mise à jour** : Si ID Bubble existant
   - **Suppression** : Si `actif = false` et ID Bubble existant

6. **💾 Sauvegarde PostgreSQL**
   - Table cible : `directories` (ou `directories_staging` en test)
   - Upsert basé sur `temp_slug`
   - Stockage de l'ID Bubble (`_id`)

### 📊 Structure des données

| Champ | Description | Source |
|-------|-------------|--------|
| id | ID unique (user_id ou agency_id) | Calculé |
| annuaire | Flag pour différencier agents/agences | Calculé |
| nom | Nom affiché | User ou agence |
| nom_agence | Nom de l'agence | Table agences |
| adresse | Adresse (vide pour agents sauf indépendants) | Conditionnel |
| telephone | Téléphone principal | Mobile, fixe ou secondaire |
| presentation | Présentation courte | User ou agence |
| actif | Statut d'activation | activated ou !supprime |
| image | URL de l'image principale | Netty ou agence |
| mignature | URL de la miniature | Priorité : mignature > image |
| google_my_business | URL GMB | Table agences |
| monday-sunday | Horaires d'ouverture | Table agences |
| note_moyenne | Note moyenne Google | Table agences |
| order | Ordre d'affichage | Nombre de biens (+10000 pour agences) |
| vente | Nombre de biens en vente | Calculé |
| loc | Nombre de biens en location | Calculé |
| temp_slug | Identifiant unique pour l'URL | Généré |
| search | Termes de recherche | Array généré |
| _id | ID Bubble | Stocké après sync |

### 🔧 Configuration requise

- **Variables d'environnement** :
  - `DB_URL` : URL de connexion PostgreSQL
  - `BUBBLE_API_KEY` : Clé API Bubble
  - `BUBBLE_API_URL` : URL de l'API Bubble

### ⚡ Optimisations

1. **Traitement par lots** : BATCH_SIZE = 10
2. **Validation des données** : Vérification des champs requis avant traitement
3. **Gestion des erreurs** : Continue même si certains enregistrements échouent
4. **Cache de recherche** : Évite les doublons dans searchdirectory

### 🚨 Points d'attention

- Les emails internes Weinvest sont exclus de la synchronisation
- Les agences 10 et 115 sont exclues
- L'ordre d'affichage favorise les agences (+10000) par rapport aux agents individuels
- Le champ `search` nécessite un format array PostgreSQL spécial
- Le mot réservé SQL `order` est géré avec des guillemets
- SSL warnings désactivés pour les appels API

### ⚠️ Problème critique - Gestion des suppressions

{% hint style="danger" %}
**Absence de gestion automatique des suppressions**

**Problème actuel** :
- Les données de suppression ne sont pas disponibles depuis l'API Netty
- Aucun gestionnaire de suppression n'a été implémenté
- Les utilisateurs et agences supprimés restent dans le système

**Solution temporaire** :
- Suppression manuelle en cachant les enregistrements directement dans Bubble
- Processus non scalable et source d'erreurs

**Action requise** :
Construire une solution propre pour gérer les suppressions, par exemple :
- Webhook de notification depuis Netty lors des suppressions
- Interface d'administration pour marquer les suppressions
- Script de comparaison périodique pour détecter les éléments supprimés
- Soft delete avec champ `deleted_at` et processus de nettoyage
{% endhint %}

### 📈 Flux de données complet

```mermaid
graph TD
    A[Tables agences + users] -->|Requête SQL complexe| B[Données fusionnées]
    B -->|Calcul statistiques| C[Comptage biens]
    C -->|Génération slugs| D[Identifiants uniques]
    D -->|Si actif=true| E[Sync Bubble Directory]
    D -->|Toujours| F[Table directories]
    D -->|Termes recherche| G[Bubble searchdirectory]
    E -->|ID Bubble| F
```

### 📊 Schéma détaillé de la fusion des données

```mermaid
graph TB
    subgraph "Requête SQL UNION"
        A1[Branch 1: Agence ID=2<br/>Users indépendants]
        A2[Branch 2: Autres agences<br/>Users rattachés]
        A3[UNION ALL]
    end
    
    subgraph "Filtres appliqués"
        B1[Exclusions emails:<br/>laurent.lexact@<br/>stephane.moquet@<br/>etc...]
        B2[Exclusions agences:<br/>ID 10, ID 115]
        B3[Conditions:<br/>actif = true<br/>GMB renseigné ou NULL]
    end
    
    subgraph "Enrichissement données"
        C1[Comptage biens vente<br/>type_offre = 1]
        C2[Comptage biens location<br/>type_offre = 2]
        C3[Calcul order<br/>total + 10000 si agence]
    end
    
    subgraph "Génération identifiants"
        D1[Agent slug:<br/>nom-agent-{user_id}]
        D2[Agence slug:<br/>nom-agence-{agency_id}]
        D3[Search terms:<br/>[nom, agence, ville...]]
    end
    
    subgraph "Synchronisation"
        E1{actif = true?}
        E2[CREATE/UPDATE<br/>Bubble Directory]
        E3[DELETE<br/>Bubble Directory]
        E4[UPSERT<br/>directories table]
    end
    
    A1 --> A3
    A2 --> A3
    A3 --> B1
    B1 --> B2
    B2 --> B3
    
    B3 --> C1
    B3 --> C2
    C1 --> C3
    C2 --> C3
    
    C3 --> D1
    C3 --> D2
    C3 --> D3
    
    D3 --> E1
    E1 -->|Yes| E2
    E1 -->|No & has _id| E3
    E1 --> E4
    
    style A1 fill:#e8f5e9
    style A2 fill:#e8f5e9
    style E2 fill:#e1f5fe
    style E3 fill:#ffebee
```

### 📊 Structure de données du directory

```mermaid
erDiagram
    DIRECTORY {
        integer id PK "user_id ou agency_id"
        boolean annuaire "true=agence, false=agent"
        string nom "Nom affiché"
        string nom_agence "Nom de l'agence"
        string adresse "Vide pour agents sauf indép"
        string telephone "Mobile, fixe ou secondaire"
        text presentation "Description courte"
        boolean actif "activated ou !supprime"
        string image "URL image principale"
        string mignature "URL miniature"
        string google_my_business "URL GMB"
        json horaires "monday-sunday"
        float note_moyenne "Note Google"
        integer order "Nb biens + bonus agence"
        integer vente "Nb biens en vente"
        integer loc "Nb biens en location"
        string temp_slug "URL unique"
        array search "Termes recherche"
        string _id "Bubble ID"
    }
    
    USERS ||--o{ DIRECTORY : becomes
    AGENCES ||--o{ DIRECTORY : becomes
    PROPERTIES ||--o{ DIRECTORY : counted_in
    
    BUBBLE_DIRECTORY {
        string id PK
        json data "Toutes les données"
    }
    
    DIRECTORY ||--o| BUBBLE_DIRECTORY : syncs_to
```

### 🔄 Interaction avec les autres flux

Ce script Python dépend directement des données synchronisées par :
- **Workflow Agences n8n** : Pour les données des agences
- **Workflow Users n8n** : Pour les données des agents
- **Workflow Biens** : Pour le comptage des propriétés actives

Il crée un annuaire unifié qui combine toutes ces sources pour l'affichage sur le site web. 