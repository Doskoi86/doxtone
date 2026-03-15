# Architecture technique — Hearthstone Battlegrounds Clone

Ce document complète `plan_hearthstone_battlegrounds.docx` avec les specs techniques réelles alignées sur le serveur existant.

---

## 1. Protocole SignalR — État réel du serveur

Le serveur utilise SignalR (WebSocket), pas du JSON brut. Le client Unity utilisera le package SignalR .NET pour communiquer.

### 1.1 Hub endpoint

```
URL : http(s)://<host>/hubs/game
Transport : WebSocket (fallback LongPolling)
Auth : JWT Bearer (bypass en dev avec NoAuth: true)
```

### 1.2 Client → Serveur (Hub Methods)

| Méthode | Paramètres | Phase | Description |
|---------|-----------|-------|-------------|
| `JoinQueue` | `playerName?: string` | Lobby | Rejoint le lobby |
| `LobbyReady` | — | Lobby | Se déclare prêt |
| `LeaveQueue` | — | Lobby | Quitte le lobby |
| `SelectHero` | `gameId, heroId` | Héros | Choisit un héros parmi les offres |
| `BuyMinion` | `gameId, offerIndex` | Recrutement | Achète un minion (3 gold) |
| `SellMinion` | `gameId, instanceIdStr` | Recrutement | Vend un minion (+1 gold) |
| `PlayMinion` | `gameId, instanceIdStr, slotIndex` | Recrutement | Pose un minion main → board (-1 = auto) |
| `MoveMinion` | `gameId, instanceIdStr, toAliveIndex` | Recrutement | Réorganise le board |
| `RefreshShop` | `gameId` | Recrutement | Reroll le shop (-1 gold) |
| `FreezeShop` | `gameId` | Recrutement | Gèle le shop |
| `UpgradeTavern` | `gameId` | Recrutement | Monte le tier de taverne |
| `ReadyForCombat` | `gameId` | Recrutement | Prêt pour le combat |
| `AnimationDone` | `gameId` | Résultats | Animations terminées côté client |

### 1.3 Serveur → Client (Events)

#### Connexion & Lobby

| Event | Payload | Destinataire |
|-------|---------|-------------|
| `Connected` | `playerId, playerName` | Appelant |
| `LobbyJoined` | `players[{Id,Name}], readyIds[]` | Appelant |
| `LobbyPlayerJoined` | `playerId, name` | Autres joueurs lobby |
| `LobbyPlayerReady` | `playerId` | Tout le lobby |
| `LobbyPlayerLeft` | `playerId` | Tout le lobby |
| `QueueLeft` | — | Appelant |

#### Initialisation partie

| Event | Payload | Destinataire |
|-------|---------|-------------|
| `GameFound` | `gameId, players[{Id,Name}]` | Tous les joueurs matchés |
| `HeroOffered` | `offers[{Id,Name,Description}]` | Chaque joueur (offres uniques) |
| `HeroSelected` | `heroId` | Appelant |
| `AllHeroesSelected` | — | Tous |

#### Gestion des phases

| Event | Payload | Destinataire |
|-------|---------|-------------|
| `PhaseStarted` | `phase, turnNumber, durationSeconds` | Tous |
| `PlayersUpdated` | `players[{PlayerId,Health,Armor,TavernTier,IsEliminated}]` | Tous |

#### Shop & Board

| Event | Payload | Destinataire |
|-------|---------|-------------|
| `ShopRefreshed` | `offers[ShopOfferState], gold, upgradeCost` | Joueur concerné |
| `MinionBought` | `instanceId, name, gold` | Acheteur |
| `MinionSold` | `instanceIdStr, gold` | Vendeur |
| `ShopFrozen` | — | Joueur concerné |
| `TavernUpgraded` | `tier, gold, upgradeCost` | Joueur concerné |
| `MinionPlayed` | `instanceIdStr, slotIndex` | Joueur concerné |
| `BoardUpdated` | `minions[MinionState]` | Joueur concerné |
| `HandUpdated` | `minions[MinionState]` | Joueur concerné |
| `HeroHealthUpdated` | `health, armor` | Joueur concerné |
| `TripleFormed` | `goldenId, goldenName` | Joueur concerné |

#### Combat

| Event | Payload | Destinataire |
|-------|---------|-------------|
| `CombatReplay` | `events[CombatEventData], playerBoardSnap, opponentBoardSnap` | Les 2 combattants |
| `CombatResult` | `opponentId, didWin, damageDealt, combatLog` | Les 2 combattants |
| `PlayerReady` | `playerId` | Tous |

#### Fin de partie

| Event | Payload | Destinataire |
|-------|---------|-------------|
| `Eliminated` | `finalRank` | Joueur éliminé |
| `PlayerEliminated` | `playerId, finalRank` | Tous les restants |
| `GameWon` | — | Gagnant |
| `GameOver` | `rankings[{PlayerId,PlayerName,FinalRank}]` | Tous |
| `Error` | `message` | Appelant |

### 1.4 Structures de données

```csharp
// Minion sur le board ou en main
MinionState {
    instanceId: string (Guid),
    name: string,
    attack: int,
    health: int,
    tier: int,
    keywords: string,       // "Taunt, DivineShield" (flags enum)
    description: string,
    tribes: string,          // "Beast" ou "Mech, Beast" (comma-separated)
    isGolden: bool
}

// Offre dans le shop
ShopOfferState {
    index: int,
    instanceId: Guid,
    name: string,
    attack: int,
    health: int,
    tier: int,
    frozen: bool,
    keywords: string,
    description: string,
    tribes: string
}

// Événement de combat (pour le replay)
CombatEventData {
    type: string,            // Attack, DeathrattleTrigger, RebornTrigger, CleaveHit
    attackerId: string,
    targetId: string,
    damage: int,
    divineShieldPopped: bool,
    targetDied: bool,
    attackerDied: bool,
    attackerName: string,
    targetName: string,
    attackerAttack: int,
    targetAttack: int,
    attackerHealthAfter: int,
    targetHealthAfter: int,
    tokenName: string        // Pour Deathrattle/Reborn : nom du token invoqué
}

// Snapshot de board (avant combat)
BoardSnapshot {
    instanceId: string,
    name: string,
    attack: int,
    health: int,
    tier: int
}
```

---

## 2. Changements serveur nécessaires pour Unity 3D

Le serveur est déjà bien structuré (données logiques, pas visuelles). Les changements sont **mineurs** :

### 2.1 Changements requis (priorité haute)

| Changement | Raison | Fichier(s) |
|-----------|--------|------------|
| Ajouter `side` aux CombatEventData | Le client 3D doit savoir de quel côté l'action se passe (player/opponent) | CombatEngine.cs |
| Ajouter `slotIndex` aux CombatEventData | Pour animer le bon minion sur le board 3D | CombatEngine.cs |
| Ajouter `summonedMinion` au CombatEventData | Quand un Deathrattle invoque un token, le client a besoin des stats complètes | EffectResolver.cs |
| Ajouter event `BuffApplied` | Le client doit animer les buffs (StartOfCombat, OnAllyDeath) | EffectResolver.cs |
| Envoyer `heroId` dans PlayersUpdated | Le client affiche les portraits des héros adverses | PhaseManagerService.cs |

### 2.2 Changements souhaitables (priorité moyenne)

| Changement | Raison | Fichier(s) |
|-----------|--------|------------|
| Hero powers dans GameHub | Les 10 héros ont des pouvoirs définis mais pas câblés | GameHub.cs, nouveau HeroPowerService |
| Reconnexion automatique | Le client 3D doit pouvoir reprendre une partie après un crash | GameHub.cs, GameRoomService |
| Spectateur : board adverse | Permettre de voir le board d'un adversaire pendant le recrutement | GameHub.cs |
| Event `TurnTimer` tick | Sync précise du timer client-serveur (optionnel, le client peut compter localement) | PhaseManagerService.cs |

### 2.3 Aucun changement nécessaire

- Le shop est 100% fonctionnel (buy, sell, freeze, refresh, upgrade, triple)
- Le combat déterministe fonctionne parfaitement
- Le système de lobby est prêt
- L'élimination et le classement sont en place
- L'économie de gold est correcte

---

## 3. Architecture MockServer (client Unity)

### 3.1 Principe

Le MockServer simule **localement** les réponses du serveur pour permettre de développer et tester le client sans connexion réseau. Il implémente la même interface `IServerBridge` que le vrai client réseau.

```
┌─────────────────────────────────────────────────┐
│ Client Unity                                     │
│                                                  │
│  GameManager / UI / Combat / Shop                │
│         │                                        │
│         ▼                                        │
│  ┌─────────────────┐                             │
│  │ IServerBridge    │ ← Interface commune         │
│  └────────┬────────┘                             │
│           │                                      │
│     ┌─────┴──────┐                               │
│     ▼            ▼                               │
│  RealServer   MockServer                         │
│  Bridge       Bridge                             │
│  (SignalR)    (local)                             │
└─────────────────────────────────────────────────┘
```

### 3.2 Ce que le MockServer doit simuler

| Feature | Source de la logique | Complexité |
|---------|---------------------|------------|
| Shop (offres, achat, vente, gold) | Réimplémenter simplifié ou inclure SharedLib comme DLL | Moyenne |
| Phases et timers | Coroutines Unity | Faible |
| Combat auto | Inclure CombatEngine de SharedLib comme DLL | Faible (déjà fait) |
| Adversaires fictifs | Générer des boards aléatoires | Faible |
| Triple/Golden | Inclure TripleDetector de SharedLib | Faible |
| Héros et pouvoirs | Données JSON embarquées | Faible |
| Élimination progressive | Simuler 7 adversaires avec HP | Moyenne |

### 3.3 Réutilisation de SharedLib

**Stratégie recommandée** : compiler `AutoBattler.SharedLib` en DLL et l'inclure dans le projet Unity.

Avantages :
- Le CombatEngine, TripleDetector, BattlecryResolver, toutes les modèles sont réutilisés tels quels
- Garantit que le MockServer se comporte comme le vrai serveur
- Les données JSON (minions.json, heroes.json) sont déjà embarquées

Contrainte : SharedLib cible .NET 9. Unity utilise .NET Standard 2.1. Il faudra :
1. Ajouter `<TargetFrameworks>net9.0;netstandard2.1</TargetFrameworks>` au .csproj de SharedLib
2. Vérifier qu'aucune API .NET 9-only n'est utilisée (probablement OK)
3. Générer la DLL netstandard2.1 et la copier dans `Assets/Plugins/`

### 3.4 Interface IServerBridge

```csharp
public interface IServerBridge
{
    // Connexion
    Task ConnectAsync(string playerName);
    Task DisconnectAsync();
    bool IsConnected { get; }

    // Lobby
    Task JoinLobbyAsync();
    Task LobbyReadyAsync();
    Task LeaveLobbyAsync();

    // Sélection héros
    Task SelectHeroAsync(string heroId);

    // Actions shop
    Task BuyMinionAsync(int offerIndex);
    Task SellMinionAsync(string instanceId);
    Task PlayMinionAsync(string instanceId, int slotIndex);
    Task MoveMinionAsync(string instanceId, int toIndex);
    Task RefreshShopAsync();
    Task FreezeShopAsync();
    Task UpgradeTavernAsync();
    Task ReadyForCombatAsync();

    // Résultats
    Task AnimationDoneAsync();

    // Events serveur → client
    event Action<string, string> OnConnected;                    // playerId, playerName
    event Action<List<PlayerInfo>, List<string>> OnLobbyJoined;  // players, readyIds
    event Action<string, string> OnLobbyPlayerJoined;            // playerId, name
    event Action<string> OnLobbyPlayerReady;                     // playerId
    event Action<string> OnLobbyPlayerLeft;                      // playerId
    event Action<string, List<PlayerInfo>> OnGameFound;          // gameId, players
    event Action<List<HeroOffer>> OnHeroOffered;                 // offers
    event Action OnAllHeroesSelected;
    event Action<string, int, int> OnPhaseStarted;               // phase, turnNumber, durationSeconds
    event Action<List<PlayerPublicState>> OnPlayersUpdated;      // players
    event Action<List<ShopOfferState>, int, int> OnShopRefreshed; // offers, gold, upgradeCost
    event Action<string, string, int> OnMinionBought;            // instanceId, name, gold
    event Action<string, int> OnMinionSold;                      // instanceId, gold
    event Action OnShopFrozen;
    event Action<int, int, int> OnTavernUpgraded;                // tier, gold, upgradeCost
    event Action<string, int> OnMinionPlayed;                    // instanceId, slotIndex
    event Action<List<MinionState>> OnBoardUpdated;              // minions
    event Action<List<MinionState>> OnHandUpdated;               // minions
    event Action<int, int> OnHeroHealthUpdated;                  // health, armor
    event Action<string, string> OnTripleFormed;                 // goldenId, goldenName
    event Action<List<CombatEventData>, List<BoardSnapshot>, List<BoardSnapshot>> OnCombatReplay;
    event Action<string, bool, int, string> OnCombatResult;      // opponentId, didWin, damage, log
    event Action<int> OnEliminated;                              // finalRank
    event Action<string, int> OnPlayerEliminated;                // playerId, finalRank
    event Action OnGameWon;
    event Action<List<RankingEntry>> OnGameOver;                 // rankings
    event Action<string> OnError;                                // message
}
```

---

## 4. Flux de jeu complet — Séquence détaillée

```
CLIENT                          SERVEUR
  │                                │
  │── ConnectAsync ────────────────▶│
  │◀─────────── OnConnected ───────│
  │                                │
  │── JoinLobbyAsync ─────────────▶│
  │◀─────────── OnLobbyJoined ────│
  │◀─── OnLobbyPlayerJoined (x N) │  (autres joueurs qui rejoignent)
  │                                │
  │── LobbyReadyAsync ───────────▶│
  │◀─── OnLobbyPlayerReady (x N) ─│  (tous se déclarent prêts)
  │                                │
  │◀─────────── OnGameFound ──────│  (gameId + liste joueurs)
  │◀─────────── OnHeroOffered ────│  (2 héros au choix)
  │                                │
  │── SelectHeroAsync ────────────▶│
  │◀── OnAllHeroesSelected ───────│
  │                                │
  │◀── OnPhaseStarted("Recruiting")│  ┐
  │◀── OnPlayersUpdated ──────────│  │
  │◀── OnShopRefreshed ───────────│  │
  │                                │  │
  │── BuyMinionAsync ─────────────▶│  │  Le joueur achète, vend,
  │◀── OnMinionBought ────────────│  │  pose, déplace, reroll,
  │◀── OnHandUpdated ─────────────│  │  freeze, upgrade...
  │── PlayMinionAsync ────────────▶│  │
  │◀── OnMinionPlayed ────────────│  │  BOUCLE
  │◀── OnBoardUpdated ────────────│  │  PRINCIPALE
  │── ReadyForCombatAsync ────────▶│  │
  │                                │  │
  │◀── OnPhaseStarted("Combat") ──│  │
  │◀── OnCombatReplay ────────────│  │  → Client anime le combat
  │◀── OnCombatResult ────────────│  │
  │                                │  │
  │◀── OnPhaseStarted("Results") ─│  │
  │── AnimationDoneAsync ─────────▶│  │
  │                                │  │
  │◀── OnPhaseStarted("Recruiting")│  ┘  (tour suivant)
  │          ...                   │
  │                                │
  │◀── OnEliminated(finalRank) ───│  (si HP ≤ 0)
  │  ou                            │
  │◀── OnGameWon ─────────────────│  (si dernier debout)
  │◀── OnGameOver(rankings) ──────│
```

---

## 5. Économie et équilibrage

### Gold
- Tour 1 : 3 gold, +1/tour, max 10
- Achat : 3 gold
- Vente : 1 gold
- Reroll : 1 gold
- Upgrade : coût variable (5/7/8/9/11/12), -1/tour (min 0)

### Tavern Tier → Slots shop
| Tier | Slots | Coût upgrade | Pool par minion |
|------|-------|-------------|-----------------|
| 1 | 3 | 5 → tier 2 | 18 copies |
| 2 | 4 | 7 → tier 3 | 15 copies |
| 3 | 4 | 8 → tier 4 | 13 copies |
| 4 | 5 | 9 → tier 5 | 11 copies |
| 5 | 5 | 11 → tier 6 | 9 copies |
| 6 | 6 | — | 7 copies |

### Timer recrutement
- Tour 1-5 : 45 secondes
- Réduit de 5s tous les 5 tours
- Minimum : 15 secondes

### Dégâts au héros
- Dégâts = TavernTier du gagnant + somme des tiers des minions survivants

### MMR
| Rang | Delta MMR |
|------|-----------|
| 1er | +150 |
| 2e | +100 |
| 3e | +50 |
| 4e | 0 |
| 5e | -25 |
| 6e | -50 |
| 7e | -75 |
| 8e | -100 |

---

## 6. Contenu du jeu

### 10 Races (Tribes)
Human, Elf, Tauren, Orc, Undead, Dragon, Demon, Fairy, Mech, Beast

### 60 Minions
5 par race × 6 tiers, plus 3 tokens. Chaque minion peut avoir :
- **Battlecry** : déclenché quand posé de la main sur le board
- **Deathrattle** : déclenché à la mort
- **StartOfCombat** : déclenché au début du combat
- **OnAllyDeath** : déclenché quand un allié meurt
- **EndOfTurn** : déclenché en fin de tour de recrutement

### 6 Keywords de combat
`Taunt` (doit être attaqué en premier), `DivineShield` (absorbe 1 coup), `Poisonous` (tue en 1 coup), `Reborn` (revit à 1 HP), `Windfury` (attaque 2 fois), `Cleave` (dégâts adjacents)

### 10 Héros (1 par race)
Chaque héros a 30 HP + armure variable et un pouvoir héroïque (ActiveCost, ActiveFree, Passive, Triggered). **Les pouvoirs ne sont pas encore câblés côté serveur.**

---

## 7. Stratégie de tests

### 7.1 Tests unitaires serveur (existants : 30 tests)
- CombatEngine : 10 tests (keywords, déterminisme, dégâts)
- Shop : 14 tests (gold, achat, vente, upgrade, freeze)
- EffectResolver : 6 tests (deathrattle, startOfCombat, onAllyDeath)

### 7.2 Tests à ajouter serveur
- Hero powers (quand implémentés)
- Reconnexion
- Cas limites : 8 joueurs simultanés, ghost opponent, élimination cascade
- Protocole : vérifier les payloads envoyés au client

### 7.3 Tests client Unity
- **Scènes de test isolées** (cf. plan Phase 7.3) :
  - TestScene_Board, TestScene_CardVisual, TestScene_Shop, TestScene_Combat, TestScene_Drag, TestScene_VFX, TestScene_UI, TestScene_FullGame
- **MockServer** : chaque scène de test utilise le MockServer avec des données prédéfinies
- **Scénarios JSON** : fichiers de test reproductibles (board spécifique, shop spécifique, séquence de combat spécifique)

### 7.4 Tests E2E (client + serveur)
- Lancer le serveur en mode dev (InMemory + NoAuth)
- Le client Unity se connecte en RealServerBridge
- Test manuel d'une partie complète 2 joueurs (2 instances Unity)
- Valider : lobby → héros → shop → combat → résultats → boucle → élimination

---

## 8. Ordre de développement recommandé

Le plan docx propose 8 phases linéaires. Voici l'ordre ajusté en tenant compte des dépendances réelles :

```
PHASE 0 — Fondations Unity 3D (URP)              ← Pas de dépendance serveur
    │
PHASE 1 — IServerBridge + MockServer              ← SharedLib en DLL
    │
    ├── SERVEUR : Ajouter side/slotIndex au protocole combat
    │
PHASE 2 — Board (slots, layout)                   ← MockServer fournit les données
    │
PHASE 3 — Cartes (rendu 3D, prefab, interactions) ← Board fournit les slots
    │
PHASE 4 — Shop (achat, vente, reroll, freeze)     ← Cartes + MockServer
    │
    ├── SERVEUR : Câbler hero powers
    │
PHASE 5 — Combat (CombatSequencer, animations)    ← Cartes + MockServer + protocole enrichi
    │
PHASE 6 — VFX, Shaders, Particules                ← Peut commencer dès Phase 3
    │
PHASE 7 — UI, écrans, flux complet                ← Tout assemblé
    │
    ├── SERVEUR : Reconnexion, spectateur
    │
PHASE 8 — Polish, audio, optimisation             ← Dernière couche
    │
    └── TEST E2E : Client + Serveur réel
```
