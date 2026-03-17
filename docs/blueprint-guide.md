# Blueprint Guide — SurvivalNeeds Plugin

This guide walks through every common Blueprint task for the **SurvivalNeeds** plugin, from initial setup to responding to events and modifying values at runtime.

---

## Table of Contents

- [1. Setup](#1-setup)
  - [1a. Add the Survival Component](#1a-add-the-survival-component)
  - [1b. Create a Config Data Asset](#1b-create-a-config-data-asset)
  - [1c. Assign the Config to the Component](#1c-assign-the-config-to-the-component)
- [2. Reading Values](#2-reading-values)
  - [Health](#health)
  - [Hunger](#hunger)
  - [Thirst](#thirst)
- [3. Modifying Values](#3-modifying-values)
  - [Healing a Character](#healing-a-character)
  - [Applying Damage](#applying-damage)
  - [Feeding a Character](#feeding-a-character)
  - [Giving the Character a Drink](#giving-the-character-a-drink)
  - [Loading a Save Game](#loading-a-save-game)
- [4. Binding to Events](#4-binding-to-events)
  - [Updating the HUD](#updating-the-hud)
  - [Reacting to Death](#reacting-to-death)
  - [Low-State Warnings](#low-state-warnings)
- [5. Controlling Timers at Runtime](#5-controlling-timers-at-runtime)
- [6. Using SurvivalCharacter as a Base](#6-using-survivalcharacter-as-a-base)

---

## 1. Setup

### 1a. Add the Survival Component

1. Open your character Blueprint in the editor.
2. In the **Components** panel, click **+ Add**.
3. Search for **Survival Component** and select it.
4. The component appears in the Components list. Rename it if desired (e.g., `SurvivalComponent`).

> **Tip:** The component uses `ClassGroup = Survival` and `BlueprintSpawnableComponent`, so it always appears prominently in the component search.

### 1b. Create a Config Data Asset

1. In the **Content Browser**, navigate to the folder where you keep data assets.
2. Click **+ Add → Miscellaneous → Data Asset**.
3. In the class picker, search for **SurvivalNeedsConfig** and select it.
4. Name the asset (e.g., `DA_DefaultSurvivalConfig`).
5. Open the asset and configure each property to suit your game.

All tunable parameters — max values, drain rates, damage thresholds — live in this single asset. Refer to the [Configuration section of the README](../README.md#configuration) for a full description of each property.

### 1c. Assign the Config to the Component

1. Select the **Survival Component** in your character Blueprint.
2. In the **Details** panel, find **NorthfallGames | Needs → Config**.
3. Set it to the `SurvivalNeedsConfig` data asset you created.

The component reads from this asset at `BeginPlay` — no code changes are needed when you change config values.

---

## 2. Reading Values

All functions are on the **Survival Component** node in Blueprint. Right-click anywhere in the Event Graph and search under **NorthfallGames | Health / Hunger / Thirst** or drag off the component reference.

### Health

| Blueprint Node | Returns | Notes |
|---|---|---|
| `Get Current Health` | `float` | Absolute health value |
| `Get Current Health Percentage` | `float` (0–1) | Useful for progress bars |
| `Is Dead` | `bool` | `true` when health ≤ 0 |
| `Is Low Health` | `bool` | `true` when health ≤ `LowHealthThreshold` |
| `Is Health Regeneration Active` | `bool` | `true` when the regen timer is running |

**Example — Drive a HUD health bar:**

```
Event Tick (or bound to OnHealthChanged)
    └── SurvivalComponent → Get Current Health Percentage
            └── SetPercent (Progress Bar)
```

### Hunger

| Blueprint Node | Returns | Notes |
|---|---|---|
| `Get Current Hunger` | `float` | Absolute hunger value |
| `Get Current Hunger Percentage` | `float` (0–1) | Useful for progress bars |
| `Is Low Hunger` | `bool` | `true` when hunger ≤ `LowHungerThreshold` |
| `Is Hunger Drain Active` | `bool` | `true` when the drain timer is running |

### Thirst

| Blueprint Node | Returns | Notes |
|---|---|---|
| `Get Current Thirst` | `float` | Absolute thirst value |
| `Get Current Thirst Percentage` | `float` (0–1) | Useful for progress bars |
| `Is Low Thirst` | `bool` | `true` when thirst ≤ `LowThirstThreshold` |
| `Is Thirst Drain Active` | `bool` | `true` when the drain timer is running |

---

## 3. Modifying Values

### Healing a Character

Call **Add Health** on the Survival Component. The value is automatically clamped to `MaxHealth` and regeneration is unaffected.

```
[Use Medkit]
    └── SurvivalComponent → Add Health (Amount = 25.0)
```

### Applying Damage

Call **Apply Damage** on the Survival Component. The system tracks the damage time for the regen delay and fires `OnDeath` if health reaches `0`.

```
[On Hit]
    └── SurvivalComponent → Apply Damage (Amount = 10.0)
```

> **Note:** This is a plugin-level damage call. It is separate from Unreal's built-in `TakeDamage` / `AnyDamage` pipeline. To bridge them, call `SurvivalComponent → Apply Damage` from your character's `Event AnyDamage` handler.

### Feeding a Character

Call **Add Hunger** to restore hunger (e.g., when a food item is consumed).

```
[On Use Food Item]
    └── SurvivalComponent → Add Hunger (Amount = 30.0)
```

To reduce hunger manually (outside of the automatic drain), call **Remove Hunger**.

### Giving the Character a Drink

Call **Add Thirst** to restore thirst (e.g., when a drink item is consumed).

```
[On Use Drink Item]
    └── SurvivalComponent → Add Thirst (Amount = 40.0)
```

To reduce thirst manually, call **Remove Thirst**.

### Loading a Save Game

Use the `Initalise` family of functions when restoring values from a save file. These functions bypass the default "start at max" behaviour in `BeginPlay` and set the exact value you provide.

```
[On Load Game]
    ├── SurvivalComponent → Initalise Health  (Amount = SavedHealth)
    ├── SurvivalComponent → Initalise Hunger  (Amount = SavedHunger)
    └── SurvivalComponent → Initalise Thirst  (Amount = SavedThirst)
```

> Call `Initalise` functions **after** `BeginPlay` has run (e.g., from your Game Mode or Game Instance after the level loads). Calling them before `BeginPlay` has no effect because the systems are created inside `BeginPlay`.

---

## 4. Binding to Events

Events are bound by dragging the **Survival Component** reference into the Event Graph and selecting the delegate (e.g., `Assign On Death`). Alternatively, select the component and use the **Events** section in the Details panel.

### Updating the HUD

Bind to **On Health Changed**, **On Hunger Changed**, and **On Thirst Changed** to get push notifications whenever a value changes, rather than polling every tick.

```
Event BeginPlay
    └── SurvivalComponent → Bind Event to On Health Changed
            └── [Custom Event] HandleHealthChanged (NewHealth, HealthPercentage)
                    └── Set Percent (HealthBar, HealthPercentage)
```

Each value-changed delegate fires immediately after initialisation with the starting values, so your HUD is populated correctly from the first frame.

### Reacting to Death

Bind to **On Death** to trigger death logic:

```
SurvivalComponent → Bind Event to On Death
    └── [Custom Event] HandleDeath
            ├── Disable Input
            ├── Play Death Montage
            └── Show Respawn Screen
```

### Low-State Warnings

Bind to the **Entered** delegates to show warnings and to the **Exited** delegates to hide them:

```
SurvivalComponent → Bind Event to On Low Health Entered
    └── [Custom Event] HandleLowHealthEntered
            └── Show Low Health Vignette

SurvivalComponent → Bind Event to On Low Health Exited
    └── [Custom Event] HandleLowHealthExited
            └── Hide Low Health Vignette
```

The same pattern applies to `OnLowHungerEntered` / `OnLowHungerExited` and `OnLowThirstEntered` / `OnLowThirstExited`.

---

## 5. Controlling Timers at Runtime

You can start and stop the automatic hunger/thirst drain and health regen programmatically.

| Blueprint Node | Effect |
|---|---|
| `Start Hunger Drain System` | Begins automatic hunger loss |
| `Stop Hunger Drain System` | Pauses automatic hunger loss |
| `Start Thirst Drain System` | Begins automatic thirst loss |
| `Stop Thirst Drain System` | Pauses automatic thirst loss |
| `Start Health Passive Regeneration` | Begins passive health recovery |
| `Stop Health Passive Regeneration` | Stops passive health recovery |

**Example — Disable drain during a cutscene:**

```
On Cutscene Begin
    ├── SurvivalComponent → Stop Hunger Drain System
    └── SurvivalComponent → Stop Thirst Drain System

On Cutscene End
    ├── SurvivalComponent → Start Hunger Drain System
    └── SurvivalComponent → Start Thirst Drain System
```

---

## 6. Using SurvivalCharacter as a Base

The plugin ships `ASurvivalCharacter`, a ready-made `ACharacter` subclass that:

- Includes a `USurvivalComponent` pre-wired with automatic fall-damage forwarding.
- Sets up a third-person spring arm and camera.

To use it, reparent your character Blueprint:

1. Open your character Blueprint.
2. In the toolbar, click **Class Settings**.
3. Under **Class Options → Parent Class**, search for and select **SurvivalCharacter**.
4. Compile and save.

Your Blueprint inherits the Survival Component and all fall-damage logic automatically. You can then access the component anywhere in the Blueprint by getting **Survival Component** from `self`.

> **Tip:** If you already have a custom character that inherits from `ACharacter`, you can replicate the `SurvivalCharacter` setup by manually adding the `Survival Component` and binding `OnLanded` → `CheckForFallDamage` as shown in `SurvivalCharacter.cpp`.
