# Blood Garden

> **Souls-like action combat** — Unreal Engine 5.4 · C++ · Solo Development  
> [itch.io Page](https://kubrik.itch.io/bloodgarden) · [ArtStation](https://www.artstation.com/kubrik)

![image](https://img.itch.zone/aW1nLzI1NTE5Nzc0LmpwZw==/original/qpQ4I3.jpg)

## Overview

Blood Garden is a fast-paced souls-like action combat game built in Unreal Engine 5.4 using C++. Set in a cursed garden where corrupted flora and divine remnants coexist, the game is centered on precise stamina-driven melee combat — emphasizing timing, resource management, and mechanical mastery over raw aggression.

The combat framework is built around a stamina economy that governs attacks, dodges, and parries. Enemy AI operates on pattern-based state machines designed for readable but demanding encounters. The RPG layer provides a full equipment system with elemental damage typing, multi-slot gear, weight-based movement penalty, and a stat architecture visible across offense, defense, and elemental damage dimensions.

---

## Engine & Technical Stack

| Layer | Technology |
|---|---|
| Engine | Unreal Engine 5.4 |
| Primary Language | C++ (gameplay, AI, systems) |
| Scripting / Prototyping | Unreal Blueprint (VFX triggers, Sequencer) |
| Rendering | Lumen (dynamic GI), custom post-process materials |
| AI | Unreal Behavior Tree + custom C++ task/decorator nodes |
| Physics | Chaos — ragdoll on death, physics props |
| Animation | UE5 Animation Blueprint + `UAnimInstance` C++ subclass |
| Platform | PC (Win64/Linux) |
| 3D Pipeline | ZBrush → Maya → Substance Painter → UE5 |
| Shader Authoring | UE Material Editor + HLSL custom nodes |

---

## Architecture Overview

```
BloodGarden/
├── Source/
│   ├── Core/
│   │   ├── BGCharacter.h/.cpp                 # Player character, input handling
│   │   ├── BGGameMode.h/.cpp                  # Session, respawn, world state
│   │   └── BGPlayerController.h/.cpp          # Input mapping, HUD init
│   ├── Systems/
│   │   ├── StaminaSystem/                      # SP pool, drain, regen, gating
│   │   ├── CombatSystem/                       # Attack chains, hit detection, parry
│   │   ├── DodgeSystem/                        # I-frame dodge, directional roll
│   │   ├── EquipmentSystem/                    # Weapon/armor slots, stat derivation
│   │   ├── InventorySystem/                    # Grid inventory, sorting, weight
│   │   ├── StatSystem/                         # HP/SP/MP, offense/defense stats
│   │   ├── ElementalSystem/                    # Damage typing, status effects
│   │   ├── AIController/                       # Enemy state machines, patterns
│   │   ├── LootSystem/                         # Drop tables, pickup actors
│   │   └── SaveSystem/                         # Checkpoint, equipment persistence
│   ├── Enemies/
│   │   ├── BGEnemyBase.h/.cpp                 # Shared enemy framework
│   │   ├── CorruptedFlora/                     # Flora-type enemy variants
│   │   └── DivineRemnants/                     # Divine-type enemy variants
│   └── UI/
│       ├── HUD/                                # HP/SP/MP bars, stamina ring
│       ├── InventoryWidget/                    # Grid, tabs, sorting, DROP action
│       ├── EquipmentWidget/                    # Character panel, slot layout
│       └── StatWidget/                         # Live stat readout panel
```

---

## Core Systems: Technical Detail

### 1. Stamina System

Stamina (SP) is the central resource governing all active player actions. Every offensive and defensive action checks SP availability before executing; insufficient SP causes action failure or penalty behavior.

**SP Pool Architecture:**
- `UStaminaComponent` manages `CurrentSP`, `MaxSP`, `RegenRate`, and `RegenDelay`.
- `MaxSP` is derived from equipment and stat scaling — base value defined by character level, modified by `FStatModifierSet` from equipped gear.
- SP regenerates automatically after a configurable delay following last SP expenditure (`RegenDelay` resets on any drain event).
- SP regen is paused during active attack chains — regen only resumes on chain completion or idle.

**SP Costs (designer-configurable per action):**

| Action | SP Cost Behavior |
|---|---|
| Light Attack | Fixed cost per hit in chain |
| Heavy Attack | Higher flat cost; scales with weapon weight |
| Dodge / Roll | Fixed cost; triggers i-frame window |
| Parry | Small cost on activation; 0 cost on successful parry |
| Block (Sub slot) | Drain per damage blocked; no regen while holding |
| Art Scroll activation | Flat cost defined in scroll data asset |

**SP Exhaustion State:**
- On SP reaching 0: player enters `Exhausted` state — movement slowed, attack inputs rejected for duration.
- `bExhausted` flag gates all SP-draining actions via pre-condition check in `UCombatComponent`.
- Exit: SP regen fills to a minimum threshold, then `Exhausted` clears.

```cpp
bool UStaminaComponent::TryConsumeStamina(float Amount)
{
    if (bExhausted || CurrentSP < Amount)
    {
        if (CurrentSP <= 0.f) SetExhausted(true);
        return false;
    }

    CurrentSP = FMath::Max(0.f, CurrentSP - Amount);
    GetWorld()->GetTimerManager().SetTimer(
        RegenDelayHandle, this,
        &UStaminaComponent::BeginRegen,
        RegenDelay, false
    );
    return true;
}
```

---

### 2. Combat System
![image](https://img.itch.zone/aW1hZ2UvNDE4MDY0Mi8yNTU1NzAxOC5wbmc=/original/Jphekm.png)
Combat is structured around a light/heavy attack chain framework with stamina gating, a parry window, and a sub-slot block — all governed by `UCombatComponent`.

**Attack Chain System:**
- Light and heavy attack sequences defined as `TArray<UAnimMontage*>` per weapon type in `UWeaponDataAsset`.
- Input buffer: attack input within a configurable frame window (notify-driven) queues the next chain step.
- `AnimNotify_ComboOpen` opens the buffer; `AnimNotify_ComboClose` resets chain if no input received within window.
- Heavy attacks are branch points — pressing heavy during a light chain can diverge into a heavy finisher montage.
- Chain breaks on dodge, parry, or SP exhaustion.

**Hit Detection:**
- Per-attack hitbox defined in `FHitboxConfig` per montage: shape (capsule/box/sphere), offset, extent, and active frame range.
- Swept trace executed each frame within the active window via `UKismetSystemLibrary::SweepMultiByChannel`.
- Hit registry prevents the same actor from being hit multiple times per swing via a `TSet<AActor*> HitActorsThisSwing`, cleared at chain end.
- Hit resolution: `UGameplayStatics::ApplyDamage` → enemy `TakeDamage` override → `UCombatComponent::ResolveDamage` applies defense, elemental modifiers, crit roll, status proc check.

**Hit-Stop:**
- On confirmed hit, both attacker and target receive brief `CustomTimeDilation` reduction.
- Duration and scale configurable per attack type in `FHitStopConfig`; heavier attacks get longer, deeper hit-stop.
- Implemented via `SetCustomTimeDilation` on both actors with a `FTimerHandle` restore callback.

**Parry System:**
- Parry is a timed active input window, not a hold-block.
- `UCombatComponent::ActivateParryWindow()` opens a `bParryActive` flag for a configurable frame duration.
- On enemy attack landing within the parry window: damage negated, enemy receives full stagger, player gains a brief free-attack window (`bParryFollowUp = true`).
- Failed parry (window missed): normal damage applies, no penalty beyond SP cost already spent.
- Parry follow-up: a dedicated high-damage montage available exclusively after a successful parry, SP cost waived.

```cpp
void UCombatComponent::OnIncomingHit(float Damage, AActor* Attacker)
{
    if (bParryActive)
    {
        // Successful parry
        GetOwner()->GetComponentByClass<UStaggerComponent>()->ApplyStagger(AttackerMaxStagger);
        bParryFollowUp = true;
        return; // Negate damage
    }

    if (bBlocking)
    {
        float Blocked = Damage * BlockDamageReduction;
        StaminaComponent->TryConsumeStamina(Blocked * BlockStaminaCostRatio);
        ApplyDamage(Damage - Blocked);
        return;
    }

    ApplyDamage(Damage);
}
```

**Stagger System:**
- `UStaggerComponent` on both player and enemies tracks `CurrentStagger` against `MaxStagger`.
- Hits accumulate stagger; on threshold breach: stagger animation plays, brief control lock, stagger resets to 0.
- Stagger resistance is a stat — heavier armor increases `MaxStagger`.

---

### 3. Dodge System

Dodge is a directional roll with an i-frame window implemented in `UDodgeComponent`.

- Dodge direction sampled from player input vector at dodge initiation; defaults to backward roll if no input.
- Physics impulse applied via `UCharacterMovementComponent::AddImpulse` scaled to dodge distance stat.
- `bInvulnerable` flag set on `UCombatComponent` for the i-frame duration — all incoming damage checks this flag.
- I-frame duration configurable; scales slightly with `Dexterity`-equivalent stat (Attack Speed stat visible in HUD).
- Dodge cooldown: brief lockout after each dodge enforced via `FTimerHandle`; SP cost checked before execution.
- **Weight interaction**: total carried weight affects dodge distance and i-frame timing — heavy builds get shorter rolls with reduced i-frame coverage (see Weight System).

---

### 4. Stat System
![image](https://img.itch.zone/aW1hZ2UvNDE4MDY0Mi8yNTU1Njk5Mi5wbmc=/original/xy549w.png)

Stats are divided into **Life** and **Offense** categories as reflected in the character panel. All stat values are derived at runtime from base character level + equipment modifiers — no manually allocated points.

**Life Stats:**

| Stat | Description | Derivation |
|---|---|---|
| HP | Maximum health pool | Base + Vigor scaling + armor bonuses |
| SP | Stamina pool | Base + Endurance scaling + gear bonuses |
| SP Regen | Stamina regen rate per second | Gear + passive modifiers |
| MP | Mana pool for Art Scroll usage | Base + Intelligence scaling |
| MP Regen | Mana regen rate per second | Gear + passive modifiers |

**Offense Stats:**

| Stat | Description |
|---|---|
| Damage | Base physical damage output |
| Attack Speed | Percentage modifier on animation play rate |
| Critical Chance | Probability of crit per hit (%) |
| Critical Multiplier | Damage multiplier on crit (e.g. 12×) |
| Heavy Attack Chance | Probability of triggering heavy hit proc |
| Defence Penetration | Flat reduction to target's armor value |
| Burn Damage | Elemental fire damage component |
| Freeze Damage | Elemental ice damage component |
| Shock Damage | Elemental lightning damage component |
| Bleed Damage | DoT physical damage component |

**Stat Derivation Pipeline:**
- `UStatComponent` computes final stats by summing `FBaseStatBlock` (level-scaled) + `FStatModifierSet` (equipment-contributed).
- `FStatModifierSet` is rebuilt on every equip/unequip event by iterating all active equipment slots and accumulating their `FItemStatContribution` structs.
- Stats exposed via `UStatComponent::GetFinalStat(EStatType)` — all other systems query through this interface, never accessing raw values directly.

```cpp
float UStatComponent::GetFinalStat(EStatType Type) const
{
    float Base = BaseStats.GetStat(Type);
    float Modifier = EquipmentModifiers.GetStat(Type);
    return FMath::Max(0.f, Base + Modifier);
}
```

---

### 5. Equipment System

Equipment is structured across 8 slot categories visible in the character panel:

| Slot | Key | Notes |
|---|---|---|
| Main Weapon | 1 | Primary melee weapon; defines attack chain set |
| Sub Weapon / Shield | 2 | Off-hand weapon or shield (block enabled only with shield) |
| Art Scroll | Q | Active ability item; MP cost on use |
| Skill Slots | CTRL + 1 / 2 / 3 / 4 | Consumable or active item quick slots |
| Head | — | Armor piece |
| Upper Body | — | Armor piece |
| Lower Body | — | Armor piece |
| Necklace | — | Accessory; stat bonuses |
| Bracelet | — | Accessory; stat bonuses |
| Ring | — | Accessory; often elemental or crit focused |

**Equipment Data Asset:**
- All items defined via `UItemDataAsset`: `EItemType`, `EItemRarity`, `FItemStatContribution`, `float Weight`, `USkeletalMesh* Mesh`, `UTexture2D* Icon`, `FText Description`.
- Weapon assets additionally contain: `TArray<UAnimMontage*> LightChain`, `TArray<UAnimMontage*> HeavyChain`, `FHitboxConfig HitboxPerAttack[]`, `EWeaponType`, `float SPCostLight`, `float SPCostHeavy`.

**Equip / Unequip Flow:**
- Player selects item in inventory → `UEquipmentComponent::TryEquip(UItemDataAsset*, EEquipSlot)`.
- Type compatibility checked via `EItemType` vs slot requirements.
- On success: previous item returned to inventory, new item equipped, `RebuildStatModifiers()` called, `OnEquipmentChanged` delegate broadcast.
- UI: character model mesh components swap to equipped asset meshes; `USkeletalMeshComponent::SetSkeletalMesh` per slot.

**Shield / Sub Slot Logic:**
- `bBlockEnabled` on `UCombatComponent` is only true when `SubSlot` contains an item with `EItemType::Shield`.
- One-handed weapon check enforced: two-handed weapons set `bTwoHanded = true` on `UEquipmentComponent`, which locks the Sub slot and clears it if occupied.

---

### 6. Inventory System

Inventory is a grid-based container with filtering, multi-column sorting, and a direct DROP action.

**Grid Architecture:**
- `UInventoryComponent` stores items as `TArray<FInventorySlot>` — each slot holds a `UItemDataAsset*` and `int32 StackCount`.
- Grid dimensions are fixed (5-column layout visible in UI); slot count configurable per character.
- Stacking: stackable items (consumables) increment `StackCount`; non-stackable items (weapons, armor) occupy individual slots.

**Weight System:**
- Each item has `float Weight` in its data asset.
- `UInventoryComponent` tracks `CurrentWeight` and `MaxWeight` (shown as `53.9 / 300` in UI).
- Exceeding `MaxWeight` triggers an overencumbered state: movement speed penalty applied, dodge distance reduced.
- Weight is a meaningful decision axis — players manage inventory load against stat benefits.

**Sorting:**
- Four sort modes: `Type`, `Rarity`, `Value`, `Weight` — visible as sort buttons in inventory footer.
- Sort implemented via `TArray::Sort` with a lambda comparator selecting the active `ESortMode`.
- Sorting is non-destructive — original pickup order preserved in a separate `PickupOrderIndex` on `FInventorySlot` for reset.

**Tabs:**
- Inventory panel supports category tabs (armor/weapons, skills, artifacts, books) driven by `EInventoryTab` enum.
- Active tab filters the displayed slots via `TArray<FInventorySlot*> GetFilteredSlots(EInventoryTab)`.

**DROP Action:**
- Selecting an item and confirming DROP spawns a `AItemPickupActor` at the player's feet and removes the item from inventory.
- World pickup actor uses a `UStaticMeshComponent` with the item's display mesh and a `USphereComponent` overlap for re-pickup.

---

### 7. Elemental Damage & Status Effects

Eight elemental/damage channels contribute independently to total DPS, allowing diverse build archetypes.

**Damage Channels:**

| Channel | Status Effect | Effect Description |
|---|---|---|
| Physical (base) | — | Standard damage, mitigated by armor |
| Burn | Burning | DoT fire damage per tick |
| Freeze | Frozen | Movement slow, attack speed reduction |
| Shock | Shocked | Interrupt chance on hit |
| Bleed | Bleeding | DoT physical damage scaling with attacker's Bleed stat |

**Status Effect Architecture:**
- `UStatusEffectComponent` on all `ABGEnemyBase` and `ABGCharacter` instances.
- Each effect is a `FStatusEffectData` struct: `EStatusType`, `float DamagePerTick`, `float Duration`, `float TickRate`, `float StackCount`.
- Effects stack to configurable maximums (e.g. Bleed stacks ×5); each stack increments `DamagePerTick`.
- Application: `UCombatComponent::ResolveDamage` rolls against `Chance` values from attacker's `FStatModifierSet`; on proc, calls `UStatusEffectComponent::ApplyEffect`.
- Tick managed via `FTimerHandle` per active effect; on expiry, effect is removed from the active set.

**Defence Penetration:**
- `DefencePenetration` stat on attacker is subtracted from target's effective armor before physical damage mitigation calculation.
- Effective armor floored at 0 — penetration cannot invert armor into damage amplification.

---

### 8. Art Scroll System
![image](https://img.itch.zone/aW1hZ2UvNDE4MDY0Mi8yNTU1NzAzMC5wbmc=/original/2gIN0I.png)

Art Scrolls occupy the dedicated `A` slot and provide an active ability triggered via a distinct input binding.

- Each scroll is a `UArtScrollDataAsset`: `float MPCost`, `float Cooldown`, `UAnimMontage* CastMontage`, `TSubclassOf<UArtEffectBase> Effect`.
- `UArtEffectBase` is the base class for all scroll effects — subclasses implement `Activate(AActor* Instigator)`.
- MP cost deducted on activation; insufficient MP blocks activation.
- Cooldown enforced via `FTimerHandle` on `UArtScrollComponent`; UI cooldown overlay driven by normalized remaining time.
- Examples: AoE burst centered on player, targeted projectile, brief self-buff (attack speed, damage amp).

---

### 9. Enemy AI System
![image](https://img.itch.zone/aW1hZ2UvNDE4MDY0Mi8yNTU1NzAzNC5wbmc=/original/ks2YWM.png)

Enemies are `ABGEnemyBase` subclasses controlled by `ABGAIController`, using Unreal's Behavior Tree with custom C++ task and decorator nodes.

**AI State Architecture:**

```
EEnemyAIState:
  ├── Idle          → Stationary, ambient animation
  ├── Patrol        → Waypoint-based roam (optional per enemy)
  ├── Alerted       → Move to last known player position
  ├── Combat        → Active engagement — attack, advance, reposition
  ├── Staggered     → Control-locked, recovery animation
  └── Dead          → Death montage, optional ragdoll, loot drop
```

**Perception:**
- `UAIPerceptionComponent` with sight and hearing channels.
- Sight: cone-based forward detection; range and angle tuned per enemy variant via `FEnemyPerceptionConfig`.
- Hearing: player movement above speed threshold and attack impacts generate noise events.
- Line-of-sight confirmation before `Alerted` → `Combat` escalation.

**Attack Pattern System:**
- Enemy attack sequences defined in `FEnemyAttackConfig` data assets: attack type, range, wind-up duration, damage, SP-drain on block, hitbox config.
- Behavior Tree selects attacks via weighted random selection filtered by range and state conditions.
- Pattern cooldowns enforced per attack type — enemies cannot repeat the same attack consecutively if configured.
- **Telegraph system**: visual/audio cue fires on wind-up frame via `AnimNotify_AttackTelegraph` — gives player the parry/dodge window.

**Stagger Response:**
- Enemies receive stagger from player attacks via `UStaggerComponent` (shared architecture with player).
- On stagger threshold: interrupt current BT task, play stagger montage, reset stagger accumulator.
- Some elite enemies have `bStaggerResistant` — they require the stagger threshold to be exceeded in a single hit (heavy attack requirement).

**Enemy Variants:**
- `CorruptedFlora`: plant-type, mid-range attacks, slower wind-ups, higher stagger threshold.
- `DivineRemnants`: fast, aggressive, shorter telegraph windows, status effect attacks.
- Per-variant stats, attack configs, and BT subtrees reference dedicated data assets — adding new enemy types requires no code changes.

---

### 10. Loot System

- Drop tables defined in `ULootTableDataAsset`: weighted `TArray<FLootEntry>` with item reference, drop chance, min/max stack count.
- `ABGEnemyBase::OnDeath` calls `ULootManager::RollDrops(LootTable, WorldLocation)` — evaluates each entry against a random roll, spawns `AItemPickupActor` for successful drops.
- Rarity tier affects item stat ranges: items of the same type but higher rarity roll higher values within a `FStatRangeConfig` defined per rarity tier.
- World loot containers (`ALootContainerActor`) use the same `ULootTableDataAsset` reference — identical drop pipeline.

**Currency:**
- Gold (`int32 Currency`) tracked on `UInventoryComponent`; shown in HUD as the coin icon (1,000 visible in UI screenshot).
- Enemies drop gold amounts from `FGoldDropRange` on their data asset.
- Used at merchant actors for purchase and upgrade transactions.

---

### 11. Save System

- `FBGSaveData` struct: player position (checkpoint index), HP/SP/MP current values, inventory serialized as `TArray<FItemSaveEntry>`, equipped items per slot, currency, enemy kill flags (for persistent world state).
- Checkpoint objects (`ACheckpointActor`) — interact to save; also function as respawn locations.
- On death: player respawns at last activated checkpoint with full HP/SP/MP; enemies in the zone respawn (standard souls-like contract).
- Async save via `UGameplayStatics::AsyncSaveGameToSlot`.
- Items serialized by `FName ItemID` reference to data asset — assets loaded by soft reference on load to avoid hard dependencies.

---

### 12. UI Architecture

The character/inventory UI (visible in screenshot) is a composite `UUserWidget` with four primary panels:

**Character Panel (Left):**
- Weapon and skill slot widgets: `UEquipSlotWidget` instances bound to `UEquipmentComponent` delegate.
- 3D character preview: renders to `UTextureRenderTarget2D` via a dedicated `APreviewCharacterActor` with isolated scene capture — displayed as a `UImage` in the widget.
- Rotate preview: mouse drag on the preview area rotates the `APreviewCharacterActor` via yaw input captured by the widget's `NativeOnMouseMove`.

**Armor Panel (Center-Left):**
- Vertical slot list: Head, Upper, Lower, Necklace, Bracelet, Ring — each a `UArmorSlotWidget` with drag-and-drop support.

**Inventory Grid (Center):**
- Category tab bar: tab selection updates `ActiveTab` and triggers `RefreshGrid()`.
- Grid rendered as `UUniformGridPanel` populated from `UInventoryComponent::GetFilteredSlots(ActiveTab)`.
- Each cell is a `UInventorySlotWidget` — supports hover (tooltip), click-to-select, and drag-and-drop to equipment slots.
- Sort buttons at footer: each calls `UInventoryComponent::SortBy(ESortMode)` and triggers `RefreshGrid()`.
- DROP button: enabled only when a slot is selected; calls `UInventoryComponent::DropItem(SelectedSlot)`.

**Stat Panel (Right):**
- Two sections: Life and Offense stats.
- Each row is a `UStatRowWidget` bound to `UStatComponent::GetFinalStat(EStatType)`.
- Stats update live via `OnStatChanged` delegate broadcast from `UStatComponent` on any equip change.
- Scrollable — full offense stat list extends beyond visible area (Bleed, Burn, Freeze, Shock, and further damage types below fold).

---

## Performance Targets & Optimization

| Target | Approach |
|---|---|
| 60 fps (PC, 1080p+) | LOD chains on character and enemy meshes; Nanite on static environment geo |
| Animation | Linked animation layers for modular weapon-specific upper body blending |
| AI | BT updates throttled for off-screen enemies; full tick only for enemies within combat range |
| Inventory | Grid widget only renders visible slots; virtual scrolling for large inventories |
| Lumen | Hardware RT on supported GPUs; software fallback for mid-range |
| Status effects | Timer-based tick, not per-frame; inactive effects immediately cleared |
| Hit detection | Sweep only during active attack frames; disabled outside montage windows |

---

## Development Scope

| Category | Detail |
|---|---|
| Developer count | 1 (solo) |
| Engine | Unreal Engine 5.4 |
| Languages | C++, HLSL |
| 3D Assets | All original — modeled, textured, rigged, animated by developer |
| Enemy types | Multiple (Corrupted Flora, Divine Remnants, variants) |
| Elemental channels | 5 (Physical, Burn, Freeze, Shock, Bleed) |
| Equipment slots | 10 (weapon ×2, art scroll, skill ×4, armor ×3, accessories ×3) |
| Gameplay systems | 12 discrete systems (see above) |
| Platform | PC Windows / Linux |
| Development tools | UE5 Editor, ZBrush, Maya, Blender, Substance Painter, Photoshop |

---

## Related Projects

| Project | Description |
|---|---|
| [TIME SOUL](https://store.steampowered.com/app/2928270/TIME_SOUL) | Souls-like action platformer; parkour, time-as-resource, procedural gen — UE5.1 |
| [U.N. Owen Was Her](https://store.steampowered.com/app/3420540/UN_Owen_Was_Her) | Third-person horror; hunger/transformation AI, bullet-hell boss — UE5.3 |
| [Olympus of the Heavens](https://store.steampowered.com/app/3358020/Olympus_of_the_Heavens) | Isometric co-op ARPG; 12 boss gods, procedural gen, Steam co-op — UE5.3 |
| [Royal Jump](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) | Mobile platformer; touch controls, physics movement, mobile optimization |
| [ArtStation Portfolio](https://www.artstation.com/kubrik) | 3D modeling — characters, creatures, props, environments |

---

## Developer

**Kubrik** — Developer & 3D Artist  
9 years web development · 7 years 3D modeling · 5 years Unreal Engine C++  
5 shipped commercial games as sole developer.

[itch.io](https://kubrik.itch.io) · [ArtStation](https://www.artstation.com/kubrik) · [Steam](https://store.steampowered.com/search/?developer=Kubrik)

---

*All code, design, and custom assets produced by a single developer. No third-party gameplay code used in core systems.*
