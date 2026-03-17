# Survival — SurvivalNeeds Plugin

An **Unreal Engine 5** plugin that adds a fully data-driven survival needs system (health, hunger & thirst) to any game. Built and maintained by **[Northfall Games](https://x.com/NorthfallGame)**.

---

## Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Blueprint API](#blueprint-api)
  - [Health](#health)
  - [Hunger](#hunger)
  - [Thirst](#thirst)
  - [Events / Delegates](#events--delegates)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Extended Documentation](#extended-documentation)
- [License](#license)

---

## Features

| System | Highlights |
|---|---|
| ❤️ **Health** | Passive regeneration (with post-damage delay), fall damage, death state, low-health threshold |
| 🍖 **Hunger** | Timer-based drain, starvation damage when empty, start/stop at runtime |
| 💧 **Thirst** | Timer-based drain, dehydration damage when empty, start/stop at runtime |
| ⚙️ **Config** | Single `USurvivalNeedsConfig` data asset controls every parameter — no code changes needed |
| 📡 **Events** | Blueprint-assignable delegates for every state change (`OnDeath`, `OnHealthChanged`, `OnLowHungerEntered`, …) |
| 🔵 **Blueprint-first** | All public functions are `BlueprintCallable` / `BlueprintPure`; all delegates are `BlueprintAssignable` |

---

## Requirements

- **Unreal Engine 5.7**
- **Visual Studio 2022** with the components listed in `.vsconfig`:
  - *Game Development with C++* workload
  - Windows 11 SDK (22621)
  - .NET Framework 4.6.2

---

## Installation

### Using the Plugin in Your Own Project

1. Copy the `Plugins/SurvivalNeeds` folder into your project's `Plugins/` directory.
2. Re-generate Visual Studio project files (right-click the `.uproject` → *Generate Visual Studio project files*).
3. Open the project in Unreal Editor — it will compile the plugin automatically.
4. Enable the plugin via **Edit → Plugins → SurvivalNeeds** if it isn't already active.

### Running the Sample Project

1. Clone or download this repository.
2. Right-click `Survival.uproject` → **Generate Visual Studio project files**.
3. Open `Survival.sln` in Visual Studio 2022 and build (`Development Editor | Win64`).
4. Launch `Survival.uproject` — the editor opens the test map (`L_TestMap`) automatically.
5. Press **Play** to test the survival systems.

---

## Quick Start

### 1 — Add the component to your character

In **Blueprint**:
- Open your character Blueprint.
- Click **Add** and search for **Survival Component**.
- Select the component and assign a `SurvivalNeedsConfig` data asset to the **Config** property.

In **C++**:
```cpp
#include "Components/SurvivalComponent.h"

UPROPERTY(VisibleAnywhere)
TObjectPtr<USurvivalComponent> SurvivalComponent;

// In the constructor:
SurvivalComponent = CreateDefaultSubobject<USurvivalComponent>(TEXT("SurvivalComponent"));
```

### 2 — Assign a config asset

Create a `SurvivalNeedsConfig` data asset (**Content Browser → Add → Miscellaneous → Data Asset → SurvivalNeedsConfig**) and set it on the component.

### 3 — Bind to events (optional)

```blueprint
// In your character's Event Graph:
SurvivalComponent → OnDeath         → Handle death (disable input, play anim, …)
SurvivalComponent → OnHealthChanged → Update HUD health bar
SurvivalComponent → OnLowHungerEntered → Show hunger warning
```

### 4 — Use `SurvivalCharacter` (optional)

The plugin ships a ready-made `ASurvivalCharacter` that already includes:
- `USurvivalComponent` with automatic fall-damage forwarding
- Third-person camera with spring arm

Reparent your character Blueprint to `SurvivalCharacter` for an instant setup.

---

## Configuration

All tunable values live in **`USurvivalNeedsConfig`** (a `UPrimaryDataAsset`).

### Health

| Property | Default | Description |
|---|---|---|
| `MaxHealth` | `100` | Maximum health value |
| `StartRegenBeginPlay` | `true` | Enable passive regen from the start |
| `RegenerationTime` | `1 s` | Regen timer interval |
| `RegenerationRate` | `1` | Health restored per interval |
| `RegenDelay` | `5 s` | Delay after taking damage before regen resumes |
| `LowHealthThreshold` | `25` | Absolute value that triggers *low health* state |
| `bEnableFallDamage` | `true` | Toggle fall damage |
| `SafeFallDistance` | `200 u` | Max fall distance with no damage |
| `FallDamageRate` | `10` | Damage per 100 units beyond safe distance |
| `FallHeightOffset` | `20 u` | Correction offset for tick-rate variance |

### Hunger

| Property | Default | Description |
|---|---|---|
| `bHungerEnabled` | `true` | Enable/disable the hunger system entirely |
| `bStartHungerDrainOnBeginPlay` | `true` | Auto-start drain on play |
| `MaxPlayerHunger` | `100` | Maximum hunger value |
| `HungerUpdateFrequency` | `5 s` | How often hunger drains |
| `HungerDrainRate` | `0.1` | Hunger lost per update |
| `LowHungerThreshold` | `25` | Absolute value that triggers *low hunger* state |
| `bEnableStarvationDamage` | `true` | Apply damage when hunger reaches 0 |
| `StarvationDamageFrequency` | `5 s` | Starvation damage interval |
| `StarvationDamage` | `5` | Damage per starvation tick |

### Thirst

| Property | Default | Description |
|---|---|---|
| `bThirstEnabled` | `true` | Enable/disable the thirst system entirely |
| `bStartThirstDrainOnBeginPlay` | `true` | Auto-start drain on play |
| `MaxPlayerThirst` | `100` | Maximum thirst value |
| `ThirstUpdateFrequency` | `5 s` | How often thirst drains |
| `ThirstDrainRate` | `0.2` | Thirst lost per update |
| `LowThirstThreshold` | `25` | Absolute value that triggers *low thirst* state |
| `bEnableDehydrationDamage` | `true` | Apply damage when thirst reaches 0 |
| `DehydrationDamageFrequency` | `5 s` | Dehydration damage interval |
| `DehydrationDamage` | `5` | Damage per dehydration tick |

---

## Blueprint API

All functions are on the **Survival Component** (`USurvivalComponent`).

### Health

| Function | Description |
|---|---|
| `GetCurrentHealth()` | Returns current health value |
| `GetCurrentHealthPercentage()` | Returns health as 0–1 fraction |
| `IsDead()` | `true` when health ≤ 0 |
| `IsLowHealth()` | `true` when health ≤ `LowHealthThreshold` |
| `AddHealth(Amount)` | Restore health (clamped to max) |
| `ApplyDamage(Amount)` | Reduce health |
| `InitaliseHealth(Amount)` | Set health directly (use when loading a save) |
| `StartHealthPassiveRegeneration()` | Enable regen timer |
| `StopHealthPassiveRegeneration()` | Disable regen timer |
| `IsHealthRegenerationActive()` | Returns regen active state |
| `CheckForFallDamage(FallDistance)` | Manually trigger fall-damage calculation |

### Hunger

| Function | Description |
|---|---|
| `GetCurrentHunger()` | Returns current hunger value |
| `GetCurrentHungerPercentage()` | Returns hunger as 0–1 fraction |
| `IsLowHunger()` | `true` when hunger ≤ `LowHungerThreshold` |
| `IsHungerDrainActive()` | Returns drain active state |
| `AddHunger(Amount)` | Increase hunger (e.g. after eating) |
| `RemoveHunger(Amount)` | Decrease hunger manually |
| `SetHunger(Amount)` | Set hunger directly |
| `InitaliseHunger(Amount)` | Set hunger directly (use when loading a save) |
| `StartHungerDrainSystem()` | Begin automatic drain |
| `StopHungerDrainSystem()` | Pause automatic drain |

### Thirst

| Function | Description |
|---|---|
| `GetCurrentThirst()` | Returns current thirst value |
| `GetCurrentThirstPercentage()` | Returns thirst as 0–1 fraction |
| `IsLowThirst()` | `true` when thirst ≤ `LowThirstThreshold` |
| `IsThirstDrainActive()` | Returns drain active state |
| `AddThirst(Amount)` | Increase thirst (e.g. after drinking) |
| `RemoveThirst(Amount)` | Decrease thirst manually |
| `SetThirst(Amount)` | Set thirst directly |
| `InitaliseThirst(Amount)` | Set thirst directly (use when loading a save) |
| `StartThirstDrainSystem()` | Begin automatic drain |
| `StopThirstDrainSystem()` | Pause automatic drain |

### Events / Delegates

Bind these in Blueprint (or C++) to react to state changes.

| Delegate | Parameters | Fired when… |
|---|---|---|
| `OnDeath` | — | Health reaches 0 |
| `OnHealthChanged` | `float NewHealth, float HealthPercentage` | Health value changes |
| `OnLowHealthEntered` | — | Health drops below `LowHealthThreshold` |
| `OnLowHealthExited` | — | Health rises above `LowHealthThreshold` |
| `OnHungerChanged` | `float NewHunger, float HungerPercentage` | Hunger value changes |
| `OnLowHungerEntered` | — | Hunger drops below `LowHungerThreshold` |
| `OnLowHungerExited` | — | Hunger rises above `LowHungerThreshold` |
| `OnThirstChanged` | `float NewThirst, float ThirstPercentage` | Thirst value changes |
| `OnLowThirstEntered` | — | Thirst drops below `LowThirstThreshold` |
| `OnLowThirstExited` | — | Thirst rises above `LowThirstThreshold` |

---

## Architecture

```
USurvivalComponent          ← Add to any AActor
│
├── UHealthSystem           ← Regen, damage, fall damage, death
├── UHungerSystem           ← Drain, starvation damage
└── UThirstSystem           ← Drain, dehydration damage

Inheritance hierarchy (plugin-internal):
UObject
 └── UBaseNeeds             ← Config access, owner, dead flag
      └── UValueNeedsBase   ← Value clamping, percentages, add/remove/set
           ├── UHealthSystem
           └── UDrainingNeedsBase  ← Timer-based drain logic
                ├── UHungerSystem
                └── UThirstSystem

Data:
USurvivalNeedsConfig (UPrimaryDataAsset) ← All tunable parameters
```

Each system is self-contained and communicates back through delegates forwarded by `USurvivalComponent`.

---

## Project Structure

```
Survival/
├── Config/                        # Unreal Engine .ini files
├── Plugins/
│   └── SurvivalNeeds/             # The reusable plugin
│       ├── Content/               # Test map, blueprints, mannequin
│       ├── Resources/             # Plugin icon
│       ├── Source/SurvivalNeeds/
│       │   ├── Public/
│       │   │   ├── Character/     # SurvivalCharacter
│       │   │   ├── Components/    # SurvivalComponent
│       │   │   ├── Data/          # SurvivalNeedsConfig
│       │   │   ├── Needs/         # HealthSystem, HungerSystem, ThirstSystem
│       │   │   │   └── BaseClass/ # BaseNeeds, ValueNeedsBase, DrainingNeedsBase
│       │   │   └── Utils/         # SimpleFunctions
│       │   └── Private/           # Implementations
│       └── SurvivalNeeds.uplugin
├── Source/
│   └── Survival/                  # Minimal host game module
├── .vsconfig                      # Required Visual Studio components
├── Survival.uproject
└── LICENSE
```

---

## Extended Documentation

Detailed guides live in the [`docs/`](docs/) folder:

| Document | Description |
|---|---|
| [How It Works](docs/how-it-works.md) | Internal architecture, class hierarchy, startup sequence, timer flow, and delegate chain |
| [Blueprint Guide](docs/blueprint-guide.md) | Step-by-step Blueprint setup, value reads/writes, event binding, and runtime timer control |
| [C++ Extension Guide](docs/cpp-extension-guide.md) | How to add new needs in C++ using `UValueNeedsBase` and `UDrainingNeedsBase` |

---

## License

This project is released under the **MIT License** — see [LICENSE](LICENSE) for details.

© 2026 Northfall Games · [𝕏 @NorthfallGame](https://x.com/NorthfallGame)

