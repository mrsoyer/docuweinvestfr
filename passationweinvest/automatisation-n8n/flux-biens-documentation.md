# 🔄 Automatisation N8N - Flux des Biens

## 🏠 Flux Bien

### 🎯 Vue d'ensemble
Le système d'alimentation des biens fonctionne via des workflows n8n qui synchronisent les données depuis les sources externes vers la base de données Supabase.

### ⚡ Workflow Principal - Synchronisation des Biens

{% hint style="info" %}
**URL du workflow** : https://sync.weinvest.app/workflow/PWWDiTr1DRwvrh0N
{% endhint %}

#### 🔄 Fonctionnement

1. **📥 Récupération des biens en ligne**
   - Le workflow récupère tous les biens qui sont actuellement en ligne via une tâche clo
   - **Source** : Tâche clo (système externe)
   - **Destination** : Table `biens` dans la base de données

2. **🔄 Mise à jour de la table biens**
   - Les biens récupérés sont insérés/mis à jour dans la table
   - Les biens qui ne sont plus en ligne sont marqués comme "delete"
   - Cela garantit que la table reflète toujours l'état actuel des biens disponibles

3. **🗑️ Gestion des suppressions**
   - Tous les biens qui ne sont pas retournés par la tâche clo sont passés en statut "delete"
   - Cela permet de garder un historique tout en identifiant les biens qui ne sont plus actifs

{% hint style="warning" %}
**⚠️ Notes importantes**
- Ce workflow assure la synchronisation en temps réel des biens disponibles
- La logique "tout ce qui n'est pas en ligne passe en delete" garantit la cohérence des données
- La fréquence d'exécution du workflow détermine la fraîcheur des données
{% endhint %}

### 📊 Tables impactées

| Table | Action | Description |
|-------|--------|-------------|
| `biens` | INSERT/UPDATE/DELETE | Table principale des biens immobiliers |

### 🔗 Intégrations

- **Entrée** : Tâche clo (API externe)
- **Sortie** : Base de données Supabase
- **Type de flux** : Synchronisation unidirectionnelle 