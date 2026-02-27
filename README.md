# CombatGo

A lightweight ECS-based combat framework for Roblox.

---

## Architecture

CombatGo is built on an **Entity Component System (ECS)**. Game objects are represented as numeric entity IDs. Data is attached via components. Skills operate on entities by reading and writing component data.

```
Entity  →  a unique ID (number)
Component  →  a data table attached to an entity
Skill  →  logic that reads/writes components on an entity
```

---

## Setup

Place the `CombatGo` package in `ReplicatedStorage`. Require it from any Script or LocalScript:

```lua
local Combat = require(ReplicatedStorage.Combat)
```

Register entities when players spawn:

```lua
Combat.Hooks.Players.PlayerCharacterAdded:Connect(function(Player, Character)
    Combat.Entity.CreateEntity({
        [Combat.Components.Character] = { Character = Character },
        [Combat.Components.Player]    = { Player = Player },
        [Combat.Components.Health]    = { Health = 100, MaxHealth = 100 },
        [Combat.Components.Skills]    = { [1] = Combat.Skills.Fireball },
    })
end)
```

---

## Entity

```lua
local EntityId = Combat.Entity.CreateEntity(ComponentDataMap?)
Combat.Entity.DestroyEntity(EntityId)
Combat.Entity.IsValid(EntityId)  →  boolean
```

`CreateEntity` accepts an optional `{[ComponentId]: InitialData}` map. Returns a generational entity ID.

---

## Components

### Built-in components

| Name        | Fields                                                        |
|-------------|---------------------------------------------------------------|
| `Character` | `Character: Model`                                            |
| `Health`    | `Health`, `MaxHealth`, `HealthRegenAmount`, `HealthRegenRate` |
| `Shield`    | `Shield`, `MaxShield`, `ShieldRegenAmount`, `ShieldRegenRate` |
| `Player`    | `Player: Player`                                              |
| `Skills`    | `Slots: {[number]: SkillDefinition}`, `Cooldowns: {[number]: number}` |

Access ComponentIds via `Combat.Components.*`:

```lua
Combat.Components.Health   -- ComponentId
Combat.Components.Skills   -- ComponentId
```

### Replication

Character, Health, Shield, Player, and Skills components automatically replicate from server to client. The client mirrors all component data and maintains reverse lookups (e.g. `GetEntityIdFromCharacter` works on both sides).

### Custom components

```lua
local MyComponent = Combat.Component.RegisterComponent("MyComponent", function(InitialData)
    return {
        Value = InitialData and InitialData.Value or 0,
    }
end, {
    Replicated = true,           -- optional: auto-replicate to all clients
    Serialize = function(Data)   -- optional: custom serialization for replication
        return { Value = Data.Value }
    end,
    Deserialize = function(Data) -- optional: reconstruct on client
        return { Value = Data.Value }
    end,
    OnAdded = function(EntityId, Data) end,   -- optional lifecycle hooks
    OnRemoved = function(EntityId, Data) end,
})
```

### Component API

```lua
Combat.Component.AddComponent(EntityId, ComponentId, InitialData?)
Combat.Component.RemoveComponent(EntityId, ComponentId)
Combat.Component.RemoveAllComponents(EntityId)
Combat.Component.HasComponent(EntityId, ComponentId)  →  boolean
Combat.Component.GetComponentData(EntityId, ComponentId)  →  Data?
```

---

## Query

Iterate over all entities that have a set of components:

```lua
for EntityId, HealthData, CharacterData in Combat.Query.All(
    Combat.Components.Health,
    Combat.Components.Character
) do
    print(EntityId, HealthData.Health)
end
```

Iterate over all entities with a single component:

```lua
for EntityId, HealthData in Combat.Query.One(Combat.Components.Health) do
    HealthData.Health = math.min(HealthData.Health + 1, HealthData.MaxHealth)
end
```

---

## Skills

### Defining a skill

```lua
local MySkill = Combat.Skills.Create("MySkill")

-- Optional: state control
MySkill.StunStates    = { "Stunned" }           -- these effects block/cancel the skill
MySkill.PauseStates   = { "Attacking" }         -- these effects pause and auto-resume the skill
MySkill.CancelParallel = true                   -- true (default): OnCancelServer + StopServer run in parallel
                                                -- false: OnCancelServer runs first, then StopServer

MySkill.StartServer = function(ServerContext)
    ServerContext.SetCooldown(3)
    ServerContext.Message("Hit")
end

MySkill.StopServer = function(ServerContext)
    ServerContext.Message("Stop")
end

MySkill.OnCancelServer = function(ServerContext)
    -- forced interruption (StunState hit, or manual CancelSkill)
    -- runs alongside StopServer (parallel) or before it (sequential)
end

MySkill.StartClient = function(ClientContext)
    local Hit = ClientContext.MessageAwait("Hit")
    if not Hit then return end
    ClientContext.MessageAwait("Stop")
end

MySkill.StopClient = function()
    -- fires immediately on client key release
end

MySkill.OnCancelClient = function(IsPaused: boolean)
    -- IsPaused = true:  a PauseState interrupted — server will auto-resume when it clears
    -- IsPaused = false: a StunState cancelled — fully stopped
end
```

### Registering skills on an entity

Pass skills as a slot map when creating the entity:

```lua
[Combat.Components.Skills] = { [1] = Combat.Skills.MySkill, [2] = Combat.Skills.OtherSkill }
```

Or update slots at runtime:

```lua
Combat.Skills.SetSlot(EntityId, 1, Combat.Skills.MySkill)
Combat.Skills.ClearSlot(EntityId, 1)
```

### Triggering skills

**From the client** (e.g. bound to a key):
```lua
local EntityId = CharacterComponent.GetEntityIdFromCharacter(LocalPlayer.Character)
Combat.Skills.StartSkill(EntityId, 1)
```

**From the server** (e.g. forced cast):
```lua
Combat.Skills.StartSkill(NpcEntityId, 1)
```

When started from the server on a player entity, the skill first fires to the client (`StartClient` runs), then the server runs `StartServer` once the client confirms. For NPC entities, `StartServer` runs immediately.

### Cooldowns

```lua
Combat.Skills.GetCooldownRemaining(EntityId, SlotIndex)  →  number
```

Set inside `StartServer` via `ServerContext.SetCooldown(seconds)`.

### SkillDefinition fields

| Field | Type | Description |
|---|---|---|
| `StunStates` | `{string}?` | Effect names that block start and cancel mid-run |
| `PauseStates` | `{string}?` | Effect names that pause mid-run and auto-resume when cleared |
| `CancelParallel` | `boolean?` | `true` (default): `OnCancelServer` + `StopServer` run simultaneously. `false`: sequential |
| `StartServer` | `func?` | Runs on server when skill starts |
| `StopServer` | `func?` | Runs on server when skill stops (key release or `StopSkill`) |
| `OnCancelServer` | `func?` | Runs on server when forcibly interrupted by a stun or pause state |
| `StartClient` | `func?` | Runs on client when skill starts |
| `StopClient` | `func?` | Runs immediately on client when key is released |
| `OnCancelClient(IsPaused)` | `func?` | Runs on client on forced cancel. `IsPaused=true` means the server will auto-resume |

### ServerContext

| Field | Description |
|---|---|
| `EntityId` | The entity the skill is running on |
| `SlotIndex` | The slot this skill occupies |
| `Player` | The associated player, if any |
| `SetCooldown(n)` | Set cooldown to `n` seconds from now |
| `IncrementCooldown(n)` | Add `n` seconds to the current cooldown |
| `Message(str)` | Send a message to the client (resumes `MessageAwait`) |
| `Reject()` | Cancel the skill client-side (all `MessageAwait` return false) |

---

## Status Effects

### Defining an effect

```lua
local Burn = Combat.StatusEffects.Register("Burn")

Burn.OnApplied = function(EntityId, Data)
    -- Data.Damage, Data.Duration, Data.Stacks are available
end

Burn.OnTick = function(EntityId, Data, DeltaTime)
    local HealthData = Combat.Component.GetComponentData(EntityId, Combat.Components.Health)
    if HealthData then
        HealthData.Health = math.max(0, HealthData.Health - Data.Damage * DeltaTime)
    end
end

Burn.OnRemoved = function(EntityId, Data)
    -- cleanup
end

Burn.OnStack = function(EntityId, Data, ApplicationData)
    -- called instead of the default when the effect is applied while already active
    -- default (no OnStack): increments Stacks and resets Duration
    Data.Stacks += 1
    Data.Damage += ApplicationData.Damage
end
```

The tick loop and expiry run **server-side only**. The `StatusEffects` component replicates to clients automatically, so clients can read active effects.

### Applying and removing

```lua
Combat.StatusEffects.Apply(EntityId, "Burn", { Duration = 5, Damage = 10 })
Combat.StatusEffects.Remove(EntityId, "Burn")
Combat.StatusEffects.RemoveAll(EntityId)
```

`Duration = -1` (or omitted) makes an effect permanent until manually removed.

Applying an already-active effect calls `OnStack` if defined, otherwise increments `Stacks` and resets `Duration`.

### Reading effects

```lua
Combat.StatusEffects.Has(EntityId, "Burn")         →  boolean
Combat.StatusEffects.Get(EntityId, "Burn")         →  EffectData?
```

### EffectData fields

| Field | Type | Description |
|---|---|---|
| `Name` | `string` | Effect name |
| `Duration` | `number` | Remaining seconds (`-1` = permanent) |
| `Stacks` | `number` | Current stack count |
| `...` | `any` | Any fields passed in `ApplicationData` |

---

## Lookup helpers

```lua
-- Get EntityId from a Roblox Character model
CharacterComponent.GetEntityIdFromCharacter(Character)  →  EntityId?

-- Get EntityId from a Player
PlayerComponent.GetEntityIdFromPlayer(Player)  →  EntityId?

-- Get Player from EntityId
PlayerComponent.GetPlayerFromEntityId(EntityId)  →  Player?
```

---

## Packet actions (internal)

| Packet | Direction | Payload |
|---|---|---|
| `SkillAction(slot, 0)` | Client → Server | Client-initiated start |
| `SkillAction(slot, 1)` | Client → Server | Client-initiated stop |
| `SkillAction(slot, 2)` | Client → Server | Server-initiated confirm (after `SkillStart`) |
| `SkillStart(slot, name)` | Server → Client | Server requesting client start (also used for pause resume) |
| `SkillMessage(slot, msg)` | Server → Client | Resumes `MessageAwait` |
| `SkillReject(slot)` | Server → Client | Cancels skill — fires `OnCancelClient(false)` |
| `SkillPause(slot)` | Server → Client | Pauses skill — fires `OnCancelClient(true)`, server will resume |
