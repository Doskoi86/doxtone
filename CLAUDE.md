# CLAUDE.md — Hearthstone Battlegrounds Clone

Ce projet orchestre le développement d'un clone complet de Hearthstone Battlegrounds.

## Repos et emplacements

| Composant | Chemin local | GitHub | Description |
|-----------|-------------|--------|-------------|
| **Orchestration** | `D:\Perso\dev\doxTone` | (ce repo) | Plans, docs, skills, coordination |
| **Serveur (origine)** | `D:\Perso\dev\JeuCarte` | `Doskoi86/autobattler-server` | Serveur .NET 9 original (2D) |
| **Serveur (3D)** | `D:\Perso\dev\AutoBattlerServer3D` | À créer : `Doskoi86/autobattler-server-3d` | Fork serveur adapté protocole Unity 3D |
| **Client Unity 3D** | `D:\Perso\dev\AutoBattlerClient3D` | À créer : `Doskoi86/autobattler-client-3d` | Client Unity 3D (URP) |
| **Client Unity 2D (legacy)** | `D:\Perso\dev\AutoBattlerClient` | — | Ancien client 2D, référence uniquement |

## Stack technique

### Serveur (C# .NET 9)
- ASP.NET Core + SignalR (WebSocket temps réel)
- Entity Framework Core + PostgreSQL (InMemory en dev)
- JWT Bearer auth (bypass en dev avec `NoAuth: true`)
- 60 minions, 10 héros, combat déterministe seeded, shop complet
- 30 tests xUnit passent

### Client Unity 3D (à créer)
- Unity 2022.3+ LTS, Universal Render Pipeline (URP)
- DOTween (animations), Cinemachine (caméra), TextMeshPro
- SignalR client (.NET) pour la communication serveur
- Shader Graph pour les effets visuels (cartes dorées, dissolution, gel)
- Architecture : IServerBridge (MockServer local / RealServer distant)

## Commandes serveur

```bash
# Depuis D:\Perso\dev\JeuCarte (ou le fork 3D)
dotnet build AutoBattler.sln
dotnet test AutoBattler.sln
dotnet run --project AutoBattler.Server
# Serveur sur http://localhost:5192
```

## Architecture serveur

```
AutoBattler.SharedLib/   — Logique de jeu partagée
  Models/                — MinionDefinition/Instance, PlayerState, GameState, Shop, TavernPool
  Combat/                — CombatEngine (déterministe), EffectResolver, BattlecryResolver, TripleDetector
  Data/                  — minions.json (60), heroes.json (10), DataLoader

AutoBattler.Server/      — ASP.NET Core
  Hubs/GameHub.cs        — Hub SignalR (/hubs/game) : toutes les actions temps réel
  Services/              — MatchmakingService, GameRoomService, PhaseManagerService (singletons)
  Controllers/           — AuthController (JWT), GameController (REST)
  Database/              — EF Core (PlayerEntity, GameHistory, PlayerGameResult)
```

## Flux d'une partie

```
8 joueurs en queue → GameRoom créée → Sélection héros (30s)
→ Boucle de tours :
    Recrutement (45s) — actions shop via SignalR
    Combat (auto) — simulation déterministe serveur
    Résultats — dégâts appliqués, éliminations
→ Game Over → Classement + récompenses
```

## Plan de développement client (8 phases)

Détail complet dans `plan_hearthstone_battlegrounds.docx`.

| Phase | Contenu | Durée estimée |
|-------|---------|---------------|
| 0 | Fondations Unity 3D URP (caméra, post-processing, structure) | 2-3 jours |
| 1 | Architecture réseau + MockServer | 4-5 jours |
| 2 | Plateau de jeu (board, slots, drag & drop 3D) | 5-7 jours |
| 3 | Cartes : rendu 3D, données, interactions | 7-10 jours |
| 4 | Phase Shop (taverne, achat, vente, reroll, freeze, level up) | 5-7 jours |
| 5 | Phase Combat (CombatSequencer, animations automatiques) | 7-10 jours |
| 6 | VFX, Shaders, Particules | 5-7 jours |
| 7 | UI, écrans, flux de jeu complet | 5-7 jours |
| 8 | Polish, audio, optimisation | 5-7 jours |

## Conventions de développement

### Général
- **Langue** : code en anglais, commentaires et docs en français
- **Commits** : messages descriptifs en français, format `type: description`
- **Branches** : `main` (stable), `feat/*`, `fix/*`, `refactor/*`

### Serveur C#
- Tout nouveau service est singleton et injecté via DI
- Les données JSON sont des EmbeddedResource
- Le combat est déterministe : même seed → même résultat
- Les boards sont clonés avant simulation, jamais modifiés en place

### Client Unity
- **Séparation données/visuel** : ScriptableObjects pour les données, MonoBehaviour pour le rendu
- **Animations** : toujours retourner la durée ou le Tween pour le séquençage. Jamais de fire-and-forget
- **Easing** : jamais Linear sauf barres de progression. Préférer OutBack, OutQuad, InBack, OutElastic
- **Object Pooling** : ne jamais Instantiate/Destroy pendant le gameplay
- **IServerBridge** : le jeu ne sait jamais s'il parle au vrai serveur ou au MockServer

## Contexte utilisateur

Le développeur a de bonnes compétences C#/.NET mais est **débutant en Unity**. Pour toute tâche Unity :
- Fournir des instructions pas-à-pas détaillées
- Expliquer le "pourquoi" des choix Unity (Inspector vs code, Prefab workflow, etc.)
- Donner les chemins exacts dans l'éditeur Unity (menus, fenêtres, propriétés)
- Inclure des snippets de code complets et testables
