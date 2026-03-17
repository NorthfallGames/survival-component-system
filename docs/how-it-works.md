# How the SurvivalNeeds Plugin Works

This document describes the internal architecture of the **SurvivalNeeds** plugin — how the systems are created, how they communicate, and what happens at runtime.

---

## Table of Contents

- [High-Level Overview](#high-level-overview)
- [Class Hierarchy](#class-hierarchy)
- [Startup Sequence](#startup-sequence)
- [Configuration Data Asset](#configuration-data-asset)
- [Health System](#health-system)
- [Hunger System](#hunger-system)
- [Thirst System](#thirst-system)
- [Delegate Chain](#delegate-chain)
- [Death State](#death-state)
- [Starvation and Dehydration Damage](#starvation-and-dehydration-damage)

---

## High-Level Overview

The plugin is built around a single **Actor Component** — `USurvivalComponent` — that you attach to any character. When the game starts, the component creates three self-contained sub-systems (`UHealthSystem`, `UHungerSystem`, `UThirstSystem`) as `UObject` instances owned by the component.

Each sub-system manages its own state, timers, and delegates. The component acts as the single public API surface and forwards events from the sub-systems to Blueprint-assignable delegates that external code (HUDs, game modes, AI) can bind to.

All tunable numbers live in a single **data asset** (`USurvivalNeedsConfig`) that the component holds a reference to. No numbers are hard-coded in the systems — every system reads its limits and rates from this asset at runtime.

---

## Class Hierarchy

```
UObject
 └── UBaseNeeds                  — owner reference, config access, world access
      └── UValueNeedsBase         — value storage, clamping, low-threshold tracking
           ├── UHealthSystem       — regen, damage, fall damage, death
           └── UDrainingNeedsBase  — timer-driven automatic drain
                ├── UHungerSystem  — hunger drain, starvation
                └── UThirstSystem  — thirst drain, dehydration
```

| Class | Purpose |
|---|---|
| `UBaseNeeds` | Root of all needs. Holds the owning `USurvivalComponent` and provides safe getters for the config, the owner actor, the dead flag, and the world. |
| `UValueNeedsBase` | Adds a `float CurrentValue`, clamping, a `bIsLow` flag, and the full add/remove/set API. Declares five pure virtual hooks that concrete sub-classes must implement. |
| `UDrainingNeedsBase` | Adds a repeating `FTimerHandle` that automatically calls `RemoveValue(GetDrainRate())` on each tick. Declares two additional pure virtuals (`GetDrainRate`, `GetUpdateFrequency`). |
| `UHealthSystem` | Concrete value system with passive regen, fall-damage calculation, and death detection. Inherits from `UValueNeedsBase`. |
| `UHungerSystem` | Concrete draining system that reads hunger settings from the config. Inherits from `UDrainingNeedsBase`. |
| `UThirstSystem` | Concrete draining system that reads thirst settings from the config. Inherits from `UDrainingNeedsBase`. |

---

## Startup Sequence

Everything is driven from `USurvivalComponent::BeginPlay`, which calls three private helpers in order:

```
BeginPlay()
  │
  ├── CreateSystems()
  │     NewObject<UHealthSystem>  → HealthSystem
  │     NewObject<UHungerSystem>  → HungerSystem
  │     NewObject<UThirstSystem>  → ThirstSystem
  │
  ├── BindSystemDelegates()
  │     HealthSystem→OnDeath             → HandleHealthSystemDeath
  │     HealthSystem→OnHealthChanged     → HandleHealthSystemHealthChanged
  │     HungerSystem→OnHungerChanged     → HandleHungerSystemHungerChanged
  │     ThirstSystem→OnThirstChanged     → HandleThirstSystemThirstChanged
  │     … (all low-state delegates) …
  │
  └── InitialiseSystems()
        HealthSystem->Initialise(this)
          └── Sets CurrentValue = MaxHealth
              Optionally starts regen timer (StartRegenBeginPlay)
        HungerSystem->Initialise(this)
          └── Sets CurrentValue = MaxPlayerHunger
              Optionally starts drain timer (bStartHungerDrainOnBeginPlay)
        ThirstSystem->Initialise(this)
          └── Sets CurrentValue = MaxPlayerThirst
              Optionally starts drain timer (bStartThirstDrainOnBeginPlay)
```

After `InitialiseSystems`, `RefreshDerivedStats` runs once to sync any secondary state (starvation / dehydration timers) that depends on the initial values.

---

## Configuration Data Asset

`USurvivalNeedsConfig` is a `UPrimaryDataAsset`. Create one or more of these assets in the Content Browser and assign them to the **Config** property on `USurvivalComponent`.

Every system fetches its parameters through `UBaseNeeds::GetConfig()`, which simply returns `OwnerComponent->GetConfig()`. Because the config is shared across all systems, changing one asset property instantly affects every system on every component that references that asset.

See the [Configuration section of the README](../README.md#configuration) for a full table of available properties.

---

## Health System

`UHealthSystem` inherits from `UValueNeedsBase`.

**Passive regeneration** — a repeating `FTimerHandle` fires every `RegenerationTime` seconds. Each tick it checks:
1. Is the owner dead? → skip.
2. Is health already at max? → skip.
3. Has enough time passed since the last call to `ApplyDamage`? (uses `RegenDelay`) → skip if not.

If all checks pass, `AddValue(RegenerationRate)` is called.

**Fall damage** — `CheckForFallDamage(float FallDistance)` is called by `ASurvivalCharacter` whenever the character lands. The damage formula is:

```
CorrectedDistance = max(0, FallDistance - FallHeightOffset)
if CorrectedDistance <= SafeFallDistance → no damage
ExcessDistance = CorrectedDistance - SafeFallDistance
Damage = (ExcessDistance / 100.0) * FallDamageRate
```

The result is clamped to `[0, MaxHealth]` and passed to `ApplyDamage`.

**Death** — when `ApplyDamage` reduces `CurrentValue` to `≤ 0` and the owner is not already dead, `bIsDead` is set to `true`, the regen timer is stopped, and `OnDeath` is broadcast.

---

## Hunger System

`UHungerSystem` inherits from `UDrainingNeedsBase`.

On each drain tick (every `HungerUpdateFrequency` seconds), `CurrentValue` is reduced by `HungerDrainRate`. When hunger reaches `0`, the drain loop stops reducing the value further (the base class guard `if (CurrentValue <= 0) return;` prevents underflow).

When `USurvivalComponent` receives the `OnHungerChanged` event and hunger is `0` and `bEnableStarvationDamage` is `true`, it starts the **starvation timer** (see below).

---

## Thirst System

`UThirstSystem` is structurally identical to `UHungerSystem`, using `ThirstUpdateFrequency`, `ThirstDrainRate`, and `LowThirstThreshold` from the config. Reaching `0` triggers the **dehydration timer** when `bEnableDehydrationDamage` is `true`.

---

## Delegate Chain

Each sub-system fires its own typed delegates. The component binds to those delegates in `BindSystemDelegates` and re-broadcasts them through its own delegates of the same type. This means Blueprint code always binds to the component — not the individual systems — keeping the public API clean.

```
UHealthSystem::OnDeath
    → USurvivalComponent::HandleHealthSystemDeath()
        → USurvivalComponent::OnDeath.Broadcast()
            → Your Blueprint event handler

UHungerSystem::OnHungerChanged(float, float)
    → USurvivalComponent::HandleHungerSystemHungerChanged(float, float)
        → RefreshDerivedStats()           ← updates starvation timer
        → USurvivalComponent::OnHungerChanged.Broadcast(float, float)
            → Your Blueprint event handler
```

---

## Death State

Death is tracked exclusively by `UHealthSystem::bIsDead`. `UBaseNeeds::IsOwnerDead()` reads this flag through the `USurvivalComponent` so every sub-system can query it without holding a direct reference to the health system.

When the owner is dead:
- The regen timer is cleared.
- `ApplyDamage` and `AddHealth` become no-ops.
- `StartDrain` (hunger / thirst) becomes a no-op.
- Starvation and dehydration timers are stopped.

---

## Starvation and Dehydration Damage

These damage-over-time timers live on `USurvivalComponent` (not on the individual systems) because they interact with the health system:

| Timer | Trigger condition | Damage source |
|---|---|---|
| `StarvationTimerHandle` | `CurrentHunger == 0 && bEnableStarvationDamage` | `Config->StarvationDamage` applied every `StarvationDamageFrequency` seconds |
| `DehydrationTimerHandle` | `CurrentThirst == 0 && bEnableDehydrationDamage` | `Config->DehydrationDamage` applied every `DehydrationDamageFrequency` seconds |

Both timers self-cancel when the corresponding need rises above `0` or the character dies.
