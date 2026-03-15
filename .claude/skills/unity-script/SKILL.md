---
name: unity-script
description: |
  Génère des scripts Unity C# et fournit des instructions pas-à-pas pour l'éditeur Unity.
  Utiliser ce skill dès que l'utilisateur veut créer ou modifier un script Unity, un prefab,
  un ScriptableObject, un shader, un système de particules, ou tout élément du client Unity 3D.
  Aussi utiliser quand l'utilisateur demande "comment faire X dans Unity", "crée le script pour Y",
  "ajoute un composant Z", ou toute tâche liée au développement du client Hearthstone Battlegrounds.
  Ne PAS utiliser pour du code serveur C# pur (.NET/ASP.NET) qui n'implique pas Unity.
---

# Unity Script Generator — Hearthstone Battlegrounds Clone

Tu assistes un développeur **expérimenté en C#/.NET mais débutant en Unity**. Il comprend le code mais a besoin qu'on lui explique chaque interaction avec l'éditeur Unity comme si c'était la première fois.

## Contexte projet

Avant de générer quoi que ce soit, lis ces fichiers pour rester aligné :
- `D:\Perso\dev\doxTone\ARCHITECTURE.md` — protocole SignalR, structures de données, architecture MockServer
- `D:\Perso\dev\doxTone\CLAUDE.md` — conventions projet, stack technique, ordre de dev

Le projet est un clone de Hearthstone Battlegrounds :
- **Client Unity 3D** : `D:\Perso\dev\AutoBattlerClient3D` — Unity 2022.3+ LTS, URP
- **Serveur** : `D:\Perso\dev\AutoBattlerServer3D` — C# .NET 9, ASP.NET Core, SignalR
- **Packages Unity** : DOTween, Cinemachine, TextMeshPro, Input System, SignalR .NET client

## Structure des dossiers Unity

Tous les scripts vont dans `Assets/_Project/Scripts/` selon cette organisation :

```
Scripts/
├── Core/          ← GameManager, StateMachine, Events, Constants
├── Network/       ← IServerBridge, RealServerBridge, MockServerBridge, Protocol/
├── Board/         ← BoardManager, BoardSlot, BoardLayout, DragDropController
├── Cards/         ← CardData (SO), CardVisual, CardHand, CardInteraction, CardPool
├── Shop/          ← ShopManager, ShopSlot, ShopLayout, FreezeController
├── Combat/        ← CombatManager, CombatSequencer, Actions/ (ICombatAction, etc.)
├── Animation/     ← TweenPresets, AnimSequencer, ScreenShakeController
├── UI/            ← UIManager, Screens/, HUD/, Popups/, Debug/
└── Utils/         ← ObjectPool, Extensions, Helpers
```

## Règles de génération de code

### Patterns Unity obligatoires

**MonoBehaviour** — pour tout ce qui vit dans la scène (attaché à un GameObject) :
```csharp
using UnityEngine;

public class ExampleComponent : MonoBehaviour
{
    [Header("References")]
    [SerializeField] private Transform targetTransform;

    [Header("Settings")]
    [SerializeField] private float speed = 5f;

    private void Awake() { /* Cache references */ }
    private void Start() { /* Init after all Awake */ }
    private void OnDestroy() { /* Cleanup, unsubscribe events */ }
}
```

- Utiliser `[SerializeField] private` plutôt que `public` pour les champs exposés dans l'Inspector
- Grouper avec `[Header("...")]` pour la lisibilité dans l'Inspector
- Toujours nettoyer les abonnements dans `OnDestroy()`

**ScriptableObject** — pour les données (cartes, héros, config) :
```csharp
using UnityEngine;

[CreateAssetMenu(fileName = "New CardData", menuName = "AutoBattler/Card Data")]
public class CardData : ScriptableObject
{
    public string cardId;
    public string cardName;
    public int attack;
    public int health;
    // ...
}
```

**Singleton MonoBehaviour** — pour les managers (GameManager, AudioManager) :
```csharp
public class GameManager : MonoBehaviour
{
    public static GameManager Instance { get; private set; }

    private void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }
}
```

### Animations DOTween

Chaque animation DOIT retourner un `Tween` ou `Sequence` pour le séquençage. Jamais de fire-and-forget.

```csharp
// BON — retourne le tween pour chaînage
public Tween AnimateHover()
{
    return DOTween.Sequence()
        .Append(transform.DOScale(1.3f, 0.15f).SetEase(Ease.OutBack))
        .Join(transform.DOLocalMoveY(0.5f, 0.15f));
}

// MAUVAIS — fire and forget, impossible à séquencer
public void AnimateHover()
{
    transform.DOScale(1.3f, 0.15f);
    transform.DOLocalMoveY(0.5f, 0.15f);
}
```

Easing par défaut :
- Hover : `Ease.OutBack`
- Déplacement : `Ease.OutQuad`
- Disparition : `Ease.InBack`
- Pop de chiffres : `Ease.OutElastic`
- Transitions : `Ease.InOutQuad`
- JAMAIS `Ease.Linear` sauf barres de progression

### Communication serveur

Le jeu ne parle JAMAIS directement au serveur. Tout passe par `IServerBridge` :

```csharp
// BON
private IServerBridge _server;
await _server.BuyMinionAsync(offerIndex);

// MAUVAIS — couplage direct au réseau
private HubConnection _connection;
await _connection.InvokeAsync("BuyMinion", gameId, offerIndex);
```

### Object Pooling

Ne jamais `Instantiate()` / `Destroy()` pendant le gameplay. Utiliser un pool :

```csharp
var card = CardPool.Instance.Get();    // au lieu de Instantiate
CardPool.Instance.Return(card);         // au lieu de Destroy
```

## Instructions Unity Editor

Chaque fois que le script nécessite une action dans l'éditeur Unity, fournis des instructions **exactes** avec les chemins de menus. Format obligatoire :

```
📋 DANS UNITY EDITOR :
1. Dans la fenêtre Hierarchy, clic droit → Create Empty → renommer "BoardManager"
2. Dans l'Inspector (avec BoardManager sélectionné), cliquer Add Component
3. Taper "BoardManager" dans la barre de recherche → sélectionner le script
4. Dans la section "References" de l'Inspector :
   - Glisser l'objet "BoardSurface" depuis la Hierarchy vers le champ "Board Surface"
   - Glisser le prefab "BoardSlot" depuis Project (Assets/_Project/Prefabs/) vers "Slot Prefab"
5. Dans la section "Settings" :
   - Régler "Slot Count" à 7
   - Régler "Arc Radius" à 15
```

Inclure systématiquement :
- Le chemin exact du menu (File → Build Settings, Window → Package Manager, etc.)
- Les noms des fenêtres Unity (Hierarchy, Inspector, Project, Scene, Game, Console)
- Comment glisser-déposer les références (depuis quelle fenêtre vers quel champ)
- Les valeurs recommandées pour chaque paramètre exposé
- Si un prefab doit être créé : expliquer comment (GameObject → drag vers Project)

## Création de Prefabs

Quand un prefab est nécessaire, expliquer la hiérarchie complète :

```
📋 CRÉER LE PREFAB :
1. Hierarchy → clic droit → Create Empty → renommer "Card"
2. Enfant de Card : clic droit sur Card → 3D Object → Quad → renommer "CardFrame"
3. Enfant de Card : créer un autre Quad → renommer "CardArt"
4. Enfant de Card : clic droit → UI → TextMeshPro - Text → renommer "AttackText"
   (si c'est la première fois : cliquer "Import TMP Essentials" dans le popup)
5. Répéter pour "HealthText"
6. Sélectionner Card dans la Hierarchy → glisser vers Assets/_Project/Prefabs/ dans Project
7. Supprimer l'instance de la Hierarchy (le prefab est maintenant dans Project)
```

## Shader Graph

Pour les shaders, expliquer comment créer via Shader Graph :

```
📋 CRÉER UN SHADER GRAPH :
1. Dans Project : Assets/_Project/Materials/Shaders/ → clic droit
2. Create → Shader Graph → URP → Lit Shader Graph (ou Unlit selon le besoin)
3. Renommer (ex: "CardGlow")
4. Double-clic pour ouvrir l'éditeur Shader Graph
5. [Expliquer chaque node à ajouter et comment les connecter]
6. Cliquer "Save Asset" en haut à gauche
7. Pour créer un Material : clic droit dans Materials/ → Create → Material
8. Dans l'Inspector du material : Shader dropdown → choisir "Shader Graphs/CardGlow"
```

## Scènes de test

Chaque nouveau système devrait avoir une scène de test. Expliquer :

```
📋 CRÉER UNE SCÈNE DE TEST :
1. File → New Scene → Basic (Built-in)
2. File → Save As → Assets/_Project/Scenes/Test/TestScene_NomDuTest.unity
3. [Instructions pour configurer la scène avec les éléments de test]
4. Pour lancer : double-clic sur la scène dans Project, puis Play (Ctrl+P)
```

## Checklist avant de livrer un script

Avant de présenter le code final, vérifier :
- [ ] Le namespace/dossier est correct selon la structure projet
- [ ] Les `using` sont tous nécessaires (pas de using inutiles)
- [ ] Les champs Inspector ont des `[Header]` et des `[Tooltip]` si nécessaire
- [ ] Les événements sont désabonnés dans `OnDestroy()`
- [ ] Les animations retournent un `Tween`/`Sequence`
- [ ] La communication serveur passe par `IServerBridge`
- [ ] Les instructions Editor sont complètes et détaillées
- [ ] Pas de `Instantiate`/`Destroy` en gameplay (utiliser le pool)
