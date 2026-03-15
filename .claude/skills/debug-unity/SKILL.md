---
name: debug-unity
description: |
  Aide à déboguer les problèmes Unity — erreurs de compilation, logs Console, problèmes d'Inspector,
  sérialisation SignalR, NullReferenceException, prefabs cassés, shaders qui ne compilent pas,
  animations qui ne jouent pas, et tout problème lié au client Unity du projet Battlegrounds.
  Utiliser ce skill quand l'utilisateur colle une erreur Unity, un stack trace, dit "ça marche pas",
  "j'ai une erreur", "le script ne compile pas", "rien ne s'affiche", "la carte ne bouge pas",
  ou tout problème de fonctionnement du client Unity.
  Ne PAS utiliser pour du debug serveur C# pur (utiliser debug-php ou debug-api pour le serveur).
---

# Debug Unity — Hearthstone Battlegrounds Client

Tu aides à diagnostiquer et résoudre les problèmes du client Unity 3D. L'utilisateur est **débutant en Unity** mais comprend le C# — explique les concepts Unity (sérialisation, lifecycle MonoBehaviour, etc.) quand c'est pertinent pour le diagnostic.

## Contexte projet

- **Client Unity** : `D:\Perso\dev\AutoBattlerClient3D` — Unity 2022.3+ LTS, URP
- **Serveur** : `D:\Perso\dev\AutoBattlerServer3D` — C# .NET 9, SignalR
- **Packages** : DOTween, Cinemachine, TextMeshPro, SignalR .NET, Input System
- **Architecture** : `D:\Perso\dev\doxTone\ARCHITECTURE.md`

## Processus de diagnostic

### 1. Identifier le type d'erreur

Lis l'erreur ou la description du problème et classe-la :

| Catégorie | Indices | Actions |
|-----------|---------|---------|
| **Compilation** | `CS0XXX`, "cannot resolve symbol", rouge dans Console | Vérifier les `using`, les assemblies, les typos |
| **Runtime — Null** | `NullReferenceException` | Vérifier les `[SerializeField]` assignés dans l'Inspector |
| **Runtime — Réseau** | Timeout, désérialisation, `InvalidOperationException` sur SignalR | Vérifier le protocole (lancer server-sync) |
| **Visuel** | "rien ne s'affiche", "la carte est rose", shader errors | Vérifier Materials, layers, caméra, URP settings |
| **Animation** | "ne bouge pas", "animation bloquée" | Vérifier DOTween init, séquences, TimeScale |
| **Inspector** | "le champ est grisé", "je ne vois pas le composant" | Vérifier sérialisation Unity, `[SerializeField]`, types supportés |
| **Build** | Erreurs au build, assemblies manquantes | Vérifier .asmdef, packages, platform settings |

### 2. Collecte d'informations

Avant de proposer une solution, rassembler :

```
📋 INFORMATIONS NÉCESSAIRES :
1. Le message d'erreur EXACT (copier-coller depuis la Console Unity)
2. Le fichier et la ligne concernés
3. Ce que tu essayais de faire quand l'erreur est apparue
4. Est-ce que ça a déjà marché avant ? Si oui, qu'as-tu changé ?
```

Si l'utilisateur n'a pas fourni ces infos, les demander. Si le fichier source est mentionné, le lire directement.

### 3. Diagnostic par catégorie

#### Erreurs de compilation (CS0XXX)

Les plus fréquentes en Unity :

| Erreur | Cause probable | Solution |
|--------|---------------|----------|
| `CS0246` "type or namespace not found" | `using` manquant ou package pas installé | Ajouter le `using` ou installer via Package Manager |
| `CS0103` "name does not exist" | Variable non déclarée ou typo | Vérifier l'orthographe, la portée |
| `CS0029` "cannot convert type" | Types incompatibles | Cast explicite ou changer le type |
| `CS1061` "does not contain definition" | Méthode/propriété inexistante | Vérifier la version du package, l'API |
| `CS0120` "requires object reference" | Accès instance depuis static | Utiliser `Instance.Method()` ou rendre la méthode static |

**Piège Unity fréquent** : les erreurs de compilation empêchent TOUTE exécution. Si un seul script a une erreur, rien ne tourne. Vérifier la Console Unity (Window → General → Console) et résoudre TOUTES les erreurs rouges.

#### NullReferenceException

Cause n°1 des bugs Unity pour les débutants. Checklist :

1. **Référence Inspector non assignée** — Le champ `[SerializeField]` est `None` dans l'Inspector
   ```
   📋 VÉRIFIER DANS UNITY :
   1. Sélectionner le GameObject dans la Hierarchy
   2. Dans l'Inspector, trouver le composant concerné
   3. Vérifier que tous les champs marqués "(None)" sont assignés
   4. Glisser les bons objets/prefabs depuis Hierarchy ou Project
   ```

2. **GetComponent retourne null** — Le composant n'existe pas sur le GameObject
   ```csharp
   // MAUVAIS — crash silencieux si pas trouvé
   var visual = GetComponent<CardVisual>();
   visual.UpdateStats(); // NullRef si pas de CardVisual

   // BON — vérification + message clair
   if (!TryGetComponent<CardVisual>(out var visual))
   {
       Debug.LogError($"CardVisual manquant sur {gameObject.name}", this);
       return;
   }
   ```

3. **Ordre d'exécution** — `Start()` d'un script appelle un autre qui n'a pas encore fait son `Awake()`
   ```
   📋 VÉRIFIER L'ORDRE D'EXÉCUTION :
   Edit → Project Settings → Script Execution Order
   Les managers (GameManager, etc.) doivent s'exécuter en premier (valeur négative)
   ```

4. **Objet détruit** — Accès à un GameObject/Component après `Destroy()`
   ```csharp
   // Vérifier avant d'utiliser
   if (cardObject != null) cardObject.UpdateVisual();
   ```

#### Problèmes visuels

| Symptôme | Cause probable | Solution |
|----------|---------------|----------|
| Objet rose/magenta | Material manquant ou shader incompatible URP | Réassigner un material URP (pas Built-in) |
| Objet invisible | Layer non rendu par la caméra, scale à 0, position hors champ | Vérifier Layer, Transform, Camera Culling Mask |
| Pas de lumière | URP Asset pas assigné | Edit → Project Settings → Graphics → vérifier URP Asset |
| Texte TMP invisible | Pas de font asset, ou Canvas manquant pour UI | Importer TMP Essentials, vérifier le Canvas |
| Particules invisibles | Material de particule non-URP | Recréer le material avec un shader URP/Particles |

#### Problèmes DOTween

| Symptôme | Cause probable | Solution |
|----------|---------------|----------|
| Rien ne bouge | DOTween pas initialisé | Ajouter `DOTween.Init()` ou le component DOTween dans la scène |
| Animation saccadée | TimeScale à 0 | Utiliser `.SetUpdate(true)` pour ignorer le TimeScale |
| Tween qui "saute" | Tween précédent pas tué | `transform.DOKill()` avant de lancer un nouveau tween |
| Sequence bloquée | Callback qui throw une exception | Vérifier les callbacks `.OnComplete()`, `.AppendCallback()` |

#### Problèmes SignalR / Réseau

| Symptôme | Cause probable | Solution |
|----------|---------------|----------|
| Connexion refused | Serveur pas lancé ou mauvais port | Vérifier `dotnet run` + URL dans RealServerBridge |
| Timeout | Serveur bloqué ou firewall | Vérifier que le serveur répond sur `/hubs/game` |
| Données null après désérialisation | Noms de propriétés ne matchent pas | Comparer casing serveur (camelCase) vs client |
| Callback pas appelé | Pas sur le main thread | Utiliser `UnityMainThreadDispatcher` pour les callbacks SignalR |
| Event non reçu | Nom d'event différent serveur/client | Lancer le skill `server-sync` pour vérifier |

### 4. Outils de debug à suggérer

Selon le problème, recommander :

```csharp
// Debug.Log avec contexte (cliquable dans la Console → va au GameObject)
Debug.Log($"Board slot {index}: minion={minion?.Name ?? "vide"}", this);

// Debug.DrawRay pour visualiser les raycasts (visible en Scene view)
Debug.DrawRay(ray.origin, ray.direction * 100f, Color.red, 2f);

// Debug.Break pour pause l'éditeur à un moment précis
if (health <= 0) Debug.Break();
```

```
📋 ACTIVER LE MODE DEBUG :
1. Window → General → Console
2. En haut de la Console, activer les 3 icônes (Info, Warning, Error)
3. Cliquer "Collapse" pour regrouper les messages identiques
4. Double-cliquer une erreur → ouvre le fichier à la bonne ligne dans l'IDE
```

### 5. Format de réponse

Toujours structurer la réponse ainsi :

```
🔍 DIAGNOSTIC : [résumé en 1 ligne]

📌 CAUSE : [explication de pourquoi ça arrive, avec le contexte Unity si c'est un concept nouveau]

🔧 SOLUTION :
1. [étape avec chemin exact dans Unity si nécessaire]
2. [code à modifier si nécessaire]
3. [vérification que ça marche]

💡 POUR ÉVITER À L'AVENIR : [conseil préventif si pertinent]
```
