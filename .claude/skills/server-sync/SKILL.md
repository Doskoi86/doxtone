---
name: server-sync
description: |
  Vérifie la cohérence du protocole entre le client Unity et le serveur ASP.NET/SignalR.
  Utiliser ce skill quand l'utilisateur modifie un event SignalR, ajoute un hub method,
  change une structure de données (MinionState, ShopOfferState, CombatEventData, etc.),
  ou veut s'assurer que le client et le serveur sont synchronisés.
  Aussi utiliser quand l'utilisateur dit "vérifie le protocole", "est-ce que le client est à jour",
  "sync client serveur", ou signale un bug de désérialisation / données manquantes.
  Ne PAS utiliser pour du debug d'erreurs Unity non liées au réseau.
---

# Server Sync — Vérification de cohérence protocole

Ce skill analyse le protocole entre le serveur (`AutoBattlerServer3D`) et le client Unity (`AutoBattlerClient3D`) pour détecter les incohérences.

## Fichiers sources à analyser

### Côté serveur
- `D:\Perso\dev\AutoBattlerServer3D\AutoBattler.Server\Hubs\GameHub.cs` — Hub methods + events envoyés
- `D:\Perso\dev\AutoBattlerServer3D\AutoBattler.Server\Services\PhaseManagerService.cs` — Events de phase
- `D:\Perso\dev\AutoBattlerServer3D\AutoBattler.SharedLib\Models\` — Structures de données
- `D:\Perso\dev\AutoBattlerServer3D\AutoBattler.SharedLib\Combat\CombatResult.cs` — Format CombatEvent

### Côté client
- `D:\Perso\dev\AutoBattlerClient3D\Assets\_Project\Scripts\Network\IServerBridge.cs` — Interface commune
- `D:\Perso\dev\AutoBattlerClient3D\Assets\_Project\Scripts\Network\RealServerBridge.cs` — Implémentation SignalR
- `D:\Perso\dev\AutoBattlerClient3D\Assets\_Project\Scripts\Network\MockServerBridge.cs` — Implémentation mock

### Documentation
- `D:\Perso\dev\doxTone\ARCHITECTURE.md` — Référence du protocole documenté

## Processus de vérification

### Étape 1 — Inventaire des events serveur

Lire `GameHub.cs` et `PhaseManagerService.cs`. Extraire TOUS les appels `SendAsync` :
- Nom de l'event
- Paramètres (types et noms)
- Destinataire (Caller, Group, individuel via GetPlayerClient)
- Conditions de déclenchement

### Étape 2 — Inventaire des handlers client

Lire `RealServerBridge.cs` (ou `IServerBridge.cs`). Extraire TOUS les handlers `.On(...)` :
- Nom de l'event écouté
- Signature du callback (types attendus)
- Action déclenchée

### Étape 3 — Comparaison croisée

Produire un rapport avec ces sections :

```markdown
## Rapport de synchronisation protocole

### ✅ Events alignés
| Event | Serveur (types) | Client (types) | Status |
|-------|-----------------|----------------|--------|
| ShopRefreshed | (object[], int, int) | (List<ShopOffer>, int, int) | OK |

### ❌ Events manquants côté client
Events que le serveur envoie mais que le client n'écoute pas.

### ❌ Events manquants côté serveur
Events que le client attend mais que le serveur n'envoie jamais.

### ⚠️ Incohérences de types
Events où les types ne correspondent pas (ex: le serveur envoie un Guid, le client attend un string).

### ⚠️ Hub methods manquantes
Méthodes que le client appelle mais qui n'existent pas dans le Hub.

### 📋 Actions recommandées
Liste ordonnée des corrections à appliquer.
```

### Étape 4 — Vérification des structures de données

Comparer les classes/records des deux côtés :

| Structure | Serveur | Client | Champs à vérifier |
|-----------|---------|--------|-------------------|
| MinionState | anonymous type in GameHub | MinionState class | instanceId, name, attack, health, tier, keywords, description, tribes, isGolden |
| ShopOfferState | anonymous type in GameHub | ShopOffer class | index, instanceId, name, attack, health, tier, frozen, keywords, description, tribes |
| CombatEventData | CombatEvent record | CombatEventData class | type, attackerId, targetId, damage, divineShieldPopped, etc. |
| BoardSnapshot | anonymous type | BoardSnapshot class | instanceId, name, attack, health, tier |

Attention particulière aux :
- **Noms de propriétés** : le serveur utilise camelCase dans les anonymous types, vérifier que le client désérialise correctement
- **Types nullables** : `string?` vs `string`, `int?` vs `int`
- **Enums sérialisés** : le serveur envoie `Keywords.ToString()` (ex: "Taunt, DivineShield"), le client doit parser

### Étape 5 — Vérification ARCHITECTURE.md

Comparer le document d'architecture avec le code réel. Signaler toute divergence entre la documentation et l'implémentation.

## Format de sortie

Toujours terminer par un résumé exécutif :

```
📊 RÉSUMÉ : X events alignés, Y manquants côté client, Z manquants côté serveur, W incohérences de types.
Actions prioritaires : [liste courte]
```
