# C++ Extension Guide — SurvivalNeeds Plugin

This guide explains how to add new survival needs to the plugin in C++ by subclassing the two provided base classes:

- **`UValueNeedsBase`** — for needs that store a value and track a "low" threshold but do **not** drain automatically (e.g., a static stamina pool or a temperature value you control manually).
- **`UDrainingNeedsBase`** — for needs that automatically lose value over time via a repeating timer (e.g., oxygen, temperature, sanity).

Both base classes live in:

```
Plugins/SurvivalNeeds/Source/SurvivalNeeds/Public/Needs/BaseClass/
```

---

## Table of Contents

- [When to Use Each Base Class](#when-to-use-each-base-class)
- [Adding Config Properties](#adding-config-properties)
- [Example A — Static Value Need (UValueNeedsBase)](#example-a--static-value-need-uvalueneedsbase)
- [Example B — Draining Need (UDrainingNeedsBase)](#example-b--draining-need-udrainingneedsbase)
- [Registering the New System with SurvivalComponent](#registering-the-new-system-with-survivalcomponent)
- [Pure Virtual Reference](#pure-virtual-reference)

---

## When to Use Each Base Class

| Your need | Base class |
|---|---|
| Value that you add/remove manually (healing item, poison dose, …) | `UValueNeedsBase` |
| Value that decreases on its own over time (oxygen, sanity, temperature, …) | `UDrainingNeedsBase` |

`UDrainingNeedsBase` inherits from `UValueNeedsBase`, so you still have the full add/remove/set API and the low-threshold system in both cases.

---

## Adding Config Properties

Before writing the system class, add the parameters your new need requires to `USurvivalNeedsConfig` so that all values remain data-driven:

```cpp
// SurvivalNeedsConfig.h

// Stamina (UValueNeedsBase example)
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "NorthfallGames|Needs|Stamina")
float MaxPlayerStamina = 100.0f;

UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "NorthfallGames|Needs|Stamina")
float LowStaminaThreshold = 20.0f;

// Oxygen (UDrainingNeedsBase example)
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "NorthfallGames|Needs|Oxygen")
bool bOxygenEnabled = true;

UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "NorthfallGames|Needs|Oxygen")
bool bStartOxygenDrainOnBeginPlay = false;

UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "NorthfallGames|Needs|Oxygen")
float MaxPlayerOxygen = 100.0f;

UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "NorthfallGames|Needs|Oxygen")
float OxygenUpdateFrequency = 1.0f;

UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "NorthfallGames|Needs|Oxygen")
float OxygenDrainRate = 5.0f;

UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "NorthfallGames|Needs|Oxygen")
float LowOxygenThreshold = 25.0f;
```

---

## Example A — Static Value Need (UValueNeedsBase)

This example adds a **Stamina** system whose value you control entirely from gameplay code — there is no automatic drain.

### Header — `StaminaSystem.h`

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Needs/BaseClass/ValueNeedsBase.h"
#include "Components/SurvivalComponent.h"
#include "StaminaSystem.generated.h"

UCLASS()
class SURVIVALNEEDS_API UStaminaSystem : public UValueNeedsBase
{
    GENERATED_BODY()

public:

    // ---- Getters ----

    UFUNCTION(BlueprintPure, Category = "Needs|Stamina")
    float GetCurrentStamina() const { return GetCurrentValue(); }

    UFUNCTION(BlueprintPure, Category = "Needs|Stamina")
    float GetCurrentStaminaPercentage() const { return GetCurrentPercent(); }

    UFUNCTION(BlueprintPure, Category = "Needs|Stamina")
    bool IsLowStamina() const { return IsLowValue(); }

    // ---- Setters ----

    UFUNCTION(BlueprintCallable, Category = "Needs|Stamina")
    void InitaliseStamina(float Amount);

    UFUNCTION(BlueprintCallable, Category = "Needs|Stamina")
    void AddStamina(float Amount);

    UFUNCTION(BlueprintCallable, Category = "Needs|Stamina")
    void RemoveStamina(float Amount);

    UFUNCTION(BlueprintCallable, Category = "Needs|Stamina")
    void SetStamina(float NewValue);

    // ---- Delegates ----

    UPROPERTY(BlueprintAssignable, Category = "Needs|Delegates")
    FOnStaminaChanged OnStaminaChanged;           // declare in SurvivalComponent.h

    UPROPERTY(BlueprintAssignable, Category = "Needs|Delegates")
    FOnLowStaminaEntered OnLowStaminaEntered;

    UPROPERTY(BlueprintAssignable, Category = "Needs|Delegates")
    FOnLowStaminaExited OnLowStaminaExited;

protected:
    // ---- Pure virtual implementations ----

    virtual float GetMaxValue() const override;
    virtual float GetLowThreshold() const override;
    virtual void BroadcastValueChanged(float NewValue, float NewPercent) override;
    virtual void BroadcastLowEntered() override;
    virtual void BroadcastLowExited() override;
};
```

### Source — `StaminaSystem.cpp`

```cpp
#include "Needs/StaminaSystem.h"
#include "Data/SurvivalNeedsConfig.h"

void UStaminaSystem::InitaliseStamina(float Amount)
{
    InitialiseValue(Amount);
}

void UStaminaSystem::AddStamina(float Amount)
{
    AddValue(Amount);
}

void UStaminaSystem::RemoveStamina(float Amount)
{
    RemoveValue(Amount);
}

void UStaminaSystem::SetStamina(float NewValue)
{
    SetValue(NewValue);
}

float UStaminaSystem::GetMaxValue() const
{
    const USurvivalNeedsConfig* Config = GetConfig();
    return Config ? Config->MaxPlayerStamina : 0.0f;
}

float UStaminaSystem::GetLowThreshold() const
{
    const USurvivalNeedsConfig* Config = GetConfig();
    return Config ? Config->LowStaminaThreshold : 0.0f;
}

void UStaminaSystem::BroadcastValueChanged(float NewValue, float NewPercent)
{
    OnStaminaChanged.Broadcast(NewValue, NewPercent);
}

void UStaminaSystem::BroadcastLowEntered()
{
    OnLowStaminaEntered.Broadcast();
}

void UStaminaSystem::BroadcastLowExited()
{
    OnLowStaminaExited.Broadcast();
}
```

---

## Example B — Draining Need (UDrainingNeedsBase)

This example adds an **Oxygen** system that automatically drains while the character is underwater. You start and stop the drain from gameplay code; the base class handles the repeating timer for you.

### Header — `OxygenSystem.h`

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Needs/BaseClass/DrainingNeedsBase.h"
#include "Components/SurvivalComponent.h"
#include "OxygenSystem.generated.h"

UCLASS()
class SURVIVALNEEDS_API UOxygenSystem : public UDrainingNeedsBase
{
    GENERATED_BODY()

public:

    // ---- Getters ----

    UFUNCTION(BlueprintPure, Category = "Needs|Oxygen")
    float GetCurrentOxygen() const { return GetCurrentValue(); }

    UFUNCTION(BlueprintPure, Category = "Needs|Oxygen")
    float GetCurrentOxygenPercentage() const { return GetCurrentPercent(); }

    UFUNCTION(BlueprintPure, Category = "Needs|Oxygen")
    bool IsLowOxygen() const { return IsLowValue(); }

    UFUNCTION(BlueprintPure, Category = "Needs|Oxygen")
    bool IsOxygenDrainActive() const { return IsDrainActive(); }

    // ---- Setters ----

    UFUNCTION(BlueprintCallable, Category = "Needs|Oxygen")
    void InitaliseOxygen(float Amount);

    UFUNCTION(BlueprintCallable, Category = "Needs|Oxygen")
    void AddOxygen(float Amount);

    UFUNCTION(BlueprintCallable, Category = "Needs|Oxygen")
    void RemoveOxygen(float Amount);

    // ---- Drain controls (wraps the base class StartDrain / StopDrain) ----

    UFUNCTION(BlueprintCallable, Category = "Needs|Oxygen")
    void StartOxygenDrain() { StartDrain(); }

    UFUNCTION(BlueprintCallable, Category = "Needs|Oxygen")
    void StopOxygenDrain() { StopDrain(); }

    // ---- Delegates ----

    UPROPERTY(BlueprintAssignable, Category = "Needs|Delegates")
    FOnOxygenChanged OnOxygenChanged;             // declare in SurvivalComponent.h

    UPROPERTY(BlueprintAssignable, Category = "Needs|Delegates")
    FOnLowOxygenEntered OnLowOxygenEntered;

    UPROPERTY(BlueprintAssignable, Category = "Needs|Delegates")
    FOnLowOxygenExited OnLowOxygenExited;

protected:
    // ---- Pure virtual implementations ----

    virtual float GetMaxValue() const override;
    virtual float GetLowThreshold() const override;
    virtual float GetDrainRate() const override;
    virtual float GetUpdateFrequency() const override;
    virtual bool ShouldStartDrainOnBeginPlay() const override;

    virtual void BroadcastValueChanged(float NewValue, float NewPercent) override;
    virtual void BroadcastLowEntered() override;
    virtual void BroadcastLowExited() override;
};
```

### Source — `OxygenSystem.cpp`

```cpp
#include "Needs/OxygenSystem.h"
#include "Data/SurvivalNeedsConfig.h"

void UOxygenSystem::InitaliseOxygen(float Amount)
{
    InitialiseValue(Amount);
}

void UOxygenSystem::AddOxygen(float Amount)
{
    AddValue(Amount);
}

void UOxygenSystem::RemoveOxygen(float Amount)
{
    RemoveValue(Amount);
}

float UOxygenSystem::GetMaxValue() const
{
    const USurvivalNeedsConfig* Config = GetConfig();
    return Config ? Config->MaxPlayerOxygen : 0.0f;
}

float UOxygenSystem::GetLowThreshold() const
{
    const USurvivalNeedsConfig* Config = GetConfig();
    return Config ? Config->LowOxygenThreshold : 0.0f;
}

float UOxygenSystem::GetDrainRate() const
{
    const USurvivalNeedsConfig* Config = GetConfig();
    return Config ? Config->OxygenDrainRate : 0.0f;
}

float UOxygenSystem::GetUpdateFrequency() const
{
    const USurvivalNeedsConfig* Config = GetConfig();
    return Config ? Config->OxygenUpdateFrequency : 1.0f;
}

bool UOxygenSystem::ShouldStartDrainOnBeginPlay() const
{
    const USurvivalNeedsConfig* Config = GetConfig();
    return Config && Config->bOxygenEnabled && Config->bStartOxygenDrainOnBeginPlay;
}

void UOxygenSystem::BroadcastValueChanged(float NewValue, float NewPercent)
{
    OnOxygenChanged.Broadcast(NewValue, NewPercent);
}

void UOxygenSystem::BroadcastLowEntered()
{
    OnLowOxygenEntered.Broadcast();
}

void UOxygenSystem::BroadcastLowExited()
{
    OnLowOxygenExited.Broadcast();
}
```

---

## Registering the New System with SurvivalComponent

After writing your system class, wire it into `USurvivalComponent` following the same pattern used for `UHungerSystem` and `UThirstSystem`.

### 1. Add the delegate declarations (SurvivalComponent.h)

```cpp
// Delegates — Stamina
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnStaminaChanged, float, NewStamina, float, StaminaPercentage);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnLowStaminaEntered);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnLowStaminaExited);

// Delegates — Oxygen
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnOxygenChanged, float, NewOxygen, float, OxygenPercentage);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnLowOxygenEntered);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnLowOxygenExited);
```

### 2. Add the system pointer and public API (SurvivalComponent.h)

```cpp
// In the protected section:
UPROPERTY()
TObjectPtr<UStaminaSystem> StaminaSystem;

UPROPERTY()
TObjectPtr<UOxygenSystem> OxygenSystem;

// Public Blueprint-callable functions:
UFUNCTION(BlueprintCallable, Category = "NorthfallGames|Stamina")
float GetCurrentStamina() const;

UFUNCTION(BlueprintCallable, Category = "NorthfallGames|Stamina")
void AddStamina(float Amount);

UFUNCTION(BlueprintCallable, Category = "NorthfallGames|Stamina")
void RemoveStamina(float Amount);

// … and so on for Oxygen …
```

### 3. Create, bind, and initialise in SurvivalComponent.cpp

```cpp
void USurvivalComponent::CreateSystems()
{
    // Existing systems …
    HealthSystem  = NewObject<UHealthSystem>(this);
    HungerSystem  = NewObject<UHungerSystem>(this);
    ThirstSystem  = NewObject<UThirstSystem>(this);

    // New systems:
    StaminaSystem = NewObject<UStaminaSystem>(this);
    OxygenSystem  = NewObject<UOxygenSystem>(this);
}

void USurvivalComponent::BindSystemDelegates()
{
    // Existing bindings …

    if (StaminaSystem)
    {
        StaminaSystem->OnStaminaChanged.AddDynamic(this, &USurvivalComponent::HandleStaminaSystemStaminaChanged);
        StaminaSystem->OnLowStaminaEntered.AddDynamic(this, &USurvivalComponent::HandleStaminaSystemLowStaminaEntered);
        StaminaSystem->OnLowStaminaExited.AddDynamic(this, &USurvivalComponent::HandleStaminaSystemLowStaminaExited);
    }

    if (OxygenSystem)
    {
        OxygenSystem->OnOxygenChanged.AddDynamic(this, &USurvivalComponent::HandleOxygenSystemOxygenChanged);
        OxygenSystem->OnLowOxygenEntered.AddDynamic(this, &USurvivalComponent::HandleOxygenSystemLowOxygenEntered);
        OxygenSystem->OnLowOxygenExited.AddDynamic(this, &USurvivalComponent::HandleOxygenSystemLowOxygenExited);
    }
}

void USurvivalComponent::InitialiseSystems()
{
    // Existing initialisations …

    if (StaminaSystem) StaminaSystem->Initialise(this);
    if (OxygenSystem)  OxygenSystem->Initialise(this);
}
```

### 4. Implement the delegate forwarders

```cpp
void USurvivalComponent::HandleStaminaSystemStaminaChanged(float NewStamina, float StaminaPercentage)
{
    OnStaminaChanged.Broadcast(NewStamina, StaminaPercentage);
}

void USurvivalComponent::HandleStaminaSystemLowStaminaEntered()
{
    OnLowStaminaEntered.Broadcast();
}

void USurvivalComponent::HandleStaminaSystemLowStaminaExited()
{
    OnLowStaminaExited.Broadcast();
}

// … equivalent handlers for Oxygen …
```

Once these steps are complete, the new systems are fully accessible in Blueprint through the Survival Component, just like health, hunger, and thirst.

---

## Pure Virtual Reference

The tables below list every function you must implement when subclassing each base class.

### UValueNeedsBase — required overrides

| Function | Signature | What to return |
|---|---|---|
| `GetMaxValue` | `float GetMaxValue() const` | The maximum allowed value (read from config) |
| `GetLowThreshold` | `float GetLowThreshold() const` | The absolute value at or below which `bIsLow` becomes `true` (read from config) |
| `BroadcastValueChanged` | `void BroadcastValueChanged(float NewValue, float NewPercent)` | Broadcast your typed `OnXxxChanged` delegate |
| `BroadcastLowEntered` | `void BroadcastLowEntered()` | Broadcast your typed `OnLowXxxEntered` delegate |
| `BroadcastLowExited` | `void BroadcastLowExited()` | Broadcast your typed `OnLowXxxExited` delegate |

### UDrainingNeedsBase — additional required overrides

These are in addition to the five `UValueNeedsBase` overrides above.

| Function | Signature | What to return |
|---|---|---|
| `GetDrainRate` | `float GetDrainRate() const` | Amount subtracted from `CurrentValue` on each drain tick (read from config) |
| `GetUpdateFrequency` | `float GetUpdateFrequency() const` | Seconds between drain ticks — must be `> 0` (read from config) |

### UDrainingNeedsBase — optional override

| Function | Signature | Default | Override to… |
|---|---|---|---|
| `ShouldStartDrainOnBeginPlay` | `bool ShouldStartDrainOnBeginPlay() const` | `return false` | Return `true` (or check a config flag) to auto-start the drain timer during `Initialise` |
