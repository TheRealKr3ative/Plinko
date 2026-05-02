# Plinko — Live Memory Library 

[![Static Badge](https://img.shields.io/badge/build-v0.0.0--beta--(unreleased)-black)](https://github.com/TheRealKr3ative)
![Static Badge](https://img.shields.io/badge/stability-stable-green)

A live in-memory cache for Roblox player and standalone data. Clean API for defining schemas, tracking mutations, replicating state, and hooking into persistence -- without touching DataStore directly.

---

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Concepts](#core-concepts)
- [API Reference](#api-reference)
- [Replication](#replication)
- [Snapshots](#snapshots)
- [Standalone Caches](#standalone-caches)
- [Examples](#examples)
- [Contact](#contact)

---

## Features

- **Schema-first** -- declare shape and defaults upfront, keys are validated on every write
- **Dirty tracking** -- only changed fields are flushed to your persistence layer
- **Thenable hooks** -- `onLoad` and `onSave` return Promises -- you own the persistence logic
- **Replication** -- per-key visibility scoped to `"owner"` or `"all"` via Schema under the hood
- **Middleware** -- intercept and transform values before they land
- **Snapshots** -- named point-in-time copies with diff and migrate support
- **Standalone caches** -- key by anything, not just players
- **Multiple caches** -- `Plinko.define()` returns isolated instances

---

## Installation

Place Plinko and its dependencies in `ReplicatedStorage`:

```
ReplicatedStorage/
    Plinko/
        Plinko.lua
        GoodSignal.lua
        Promise.lua
        Schema.lua      -- optional, required for replication
```

Require it on both server and client:

```lua
local Plinko = require(game:GetService("ReplicatedStorage").Plinko)
```

---

## Quick Start

**Server**

```lua
local Plinko = require(game:GetService("ReplicatedStorage").Plinko)

local Players = Plinko.define("Players", {
    schema = {
        health = 100,
        kills  = 0,
        deaths = 0,
        rank   = "Bronze",
    },

    replicate = {
        health = "owner",
        kills  = "all",
        rank   = "all",
    },

    save = { interval = 60, onLeave = true },

    onLoad = function(player)
        return Promise.new(function(resolve)
            resolve(DataStore:GetAsync(player.UserId) or {})
        end)
    end,

    onSave = function(player, data)
        return Promise.new(function(resolve)
            DataStore:SetAsync(player.UserId, data)
            resolve()
        end)
    end,
})

Players:set(player, "kills", 5)
Players:increment(player, "deaths", 1)
```

**Client**

```lua
local Plinko = require(game:GetService("ReplicatedStorage").Plinko)

local Players = Plinko.define("Players", {
    schema    = { health = 100, kills = 0, deaths = 0, rank = "Bronze" },
    replicate = { health = "owner", kills = "all", rank = "all" },
})

Players:await():andThen(function(data)
    print("Loaded -- health:", data.health)
end)

Players:watch("health", function(new, old)
    HealthBar:set(new)
end)
```

---

## Core Concepts

### Cache
`Plinko.define()` returns a **Cache** -- an isolated live store with its own schema, replication config, save hooks, and signals. Multiple caches coexist independently.

### Schema
The `schema` table defines field names and their default values. Defaults are used for type validation on write and as the baseline for `:reset()` and `:migrate()`.

### Dirty tracking
Every `:set()` or `:patch()` marks affected keys dirty. Only dirty fields are passed to `onSave` -- reads and unchanged keys never trigger a flush.

### Thenables
`onLoad` and `onSave` must return a Promise. Plinko chains off whatever you return -- you choose the persistence layer.

---

## API Reference

### `Plinko.define(name, config)`

Defines a new cache and returns it.

```lua
local Cache = Plinko.define("Players", {
    schema     = { health = 100, coins = 0 },
    replicate  = { health = "owner", coins = "owner" },
    visibility = "public",
    middleware = {
        health = function(value, old)
            return math.clamp(value, 0, 100)
        end,
    },
    save       = { interval = 60, onLeave = true },
    onLoad     = function(player) return Promise.resolve({}) end,
    onSave     = function(player, data) return Promise.resolve() end,
})
```

| Field | Type | Description |
|---|---|---|
| `schema` | `table` | Field names and default values |
| `replicate` | `table?` | Per-key replication scope -- `"owner"` or `"all"` |
| `visibility` | `string?` | `"public"` (default) or `"private"` |
| `middleware` | `table?` | Per-key transform functions run before writes land |
| `save.interval` | `number?` | Auto-flush interval in seconds. `0` disables |
| `save.onLeave` | `boolean?` | Flush on `PlayerRemoving`. Defaults to `true` |
| `onLoad` | `function?` | Called on player join -- must return a Promise resolving to a data table |
| `onSave` | `function?` | Called on flush -- receives `(player, dirtyFields)`, must return a Promise |
| `standalone` | `boolean?` | Detaches from player lifecycle -- see [Standalone Caches](#standalone-caches) |

---

### Reading & Writing

```lua
Cache:get(player, key)               -- read one key
Cache:set(player, key, value)        -- write -- runs middleware, validates type, marks dirty
Cache:increment(player, key, n?)     -- numeric shorthand, n defaults to 1
Cache:patch(player, delta)           -- merge partial table -- runs middleware per key
Cache:reset(player, key)             -- revert key to schema default
Cache:flush(player)                  -- manual save -- returns thenable
Cache:loaded(player)                 -- boolean
Cache:await(player?)                 -- thenable resolving to data -- no arg on client
Cache:lock(player)                   -- freeze all writes
Cache:unlock(player)                 -- resume writes
```

---

### Visibility

```lua
Cache:hide(player)    -- make this player's cache invisible to others
Cache:show(player)    -- revert to definition visibility
```

---

### Querying

```lua
Cache:find(function(player, data)
    return data.rank == "Gold"
end)
-- returns { [Player]: data }

Cache:some(function(player, data)
    return data.health <= 0
end)
-- returns boolean

Cache:count()                                    -- total loaded entries
Cache:count(function(player, data)               -- conditional count
    return data.kills > 10
end)
```

---

### Bulk Operations

```lua
Cache:setAll(key, value)       -- set a key for every loaded entry
Cache:patchAll(delta)          -- patch every loaded entry
Cache:flushAll()               -- flush all dirty entries -- returns thenable
Cache:resetAll(key)            -- revert a key for every loaded entry
```

---

### Dirty Inspection

```lua
Cache:dirty(player)              -- { [key]: newValue } of unsaved changes
Cache:isDirty(player, key)       -- boolean for a specific key
Cache:clearDirty(player)         -- discard dirty flags without saving
```

---

### Events & Watching

```lua
-- watch a single key
Cache:watch(player, key, function(new, old) end)

-- listen to all changes on a player
Cache:onChange(player, function(key, new, old) end)

-- listen to flushes on a player
Cache:onFlushFor(player, function(dirty) end)

-- cache-level signals
Cache.onAdded:Connect(function(player, data) end)
Cache.onRemoving:Connect(function(player, data) end)
Cache.onFlush:Connect(function(player, dirty) end)
Cache.onError:Connect(function(player, err) end)
```

> On the client, `:watch()` and `:onChange()` take no player argument -- they always refer to the local player's data.

---

### Destruction

```lua
Cache:destroy()    -- flushAll -> release all entries -> fire onRemoving -> destroy signals
```

---

## Replication

Set `replicate` in the definition to control which keys reach clients.

```lua
replicate = {
    health = "owner",    -- only the owning client receives this key
    rank   = "all",      -- all clients receive this key
}
```

Replication fires automatically on every `:set()` and `:patch()` for flagged keys. Force-push manually:

```lua
Cache:push(player, key)    -- replicate one key now
Cache:pushAll(player)      -- replicate all flagged keys now
```

Set `visibility = "private"` to prevent other clients from reading `"all"` keys for a given player. Override per-player at runtime:

```lua
Cache:hide(player)
Cache:show(player)
```

Replication requires Schema. If the `replicate` field is omitted entirely, Schema is never touched.

---

## Snapshots

Named point-in-time copies of a player's data.

```lua
Cache:snapshot(player, "beforeCombat")             -- take a snapshot
Cache:getSnapshot(player, "beforeCombat")          -- read snapshot data
Cache:diff(player, "beforeCombat")                 -- { [key]: { old, new } } vs current state
Cache:migrate(player, "beforeCombat")              -- revert current state to snapshot, marks dirty
Cache:destroySnapshot(player, "beforeCombat")      -- remove one snapshot
Cache:destroySnapshots(player)                     -- remove all snapshots for player
```

Snapshots are accessible directly:

```lua
Cache._snapshots[player]["beforeCombat"]    -- raw data table
```

---

## Standalone Caches

Set `standalone = true` to decouple a cache from the player lifecycle. Useful for match state, round data, or any server-side store keyed by something other than a player.

```lua
local MatchCache = Plinko.define("MatchCache", {
    standalone = true,
    schema = {
        round      = 1,
        startTime  = 0,
        teamScores = {},
    },
})

MatchCache:create("match_001")
MatchCache:exists("match_001")
MatchCache:set("match_001", "round", 2)
MatchCache:get("match_001", "round")
MatchCache:patch("match_001", { teamScores = { red = 3, blue = 1 } })
MatchCache:release("match_001")
```

Standalone caches have no `onLoad`, `onSave`, or `PlayerRemoving` wiring. Lifecycle is fully manual.

---

## Examples

### Leaderboard query

```lua
local top = Players:find(function(_, data)
    return data.kills >= 10
end)

for player, data in pairs(top) do
    print(player.Name, data.kills)
end
```

---

### Clamp health with middleware

```lua
local Cache = Plinko.define("Players", {
    schema = { health = 100 },
    middleware = {
        health = function(value)
            return math.clamp(value, 0, 100)
        end,
    },
    ...
})

Cache:set(player, "health", 9999)
Cache:get(player, "health")    -- 100
```

---

### Combat snapshot and rollback

```lua
Cache:snapshot(player, "preRound")

-- round plays out
Cache:set(player, "health", 20)
Cache:increment(player, "deaths", 1)

-- inspect what changed
local changes = Cache:diff(player, "preRound")

-- roll back
Cache:migrate(player, "preRound")
```

---

### Round-scoped standalone cache

```lua
local MatchCache = Plinko.define("MatchCache", {
    standalone = true,
    schema = { round = 1, redScore = 0, blueScore = 0 },
})

MatchCache:create(matchId)

MatchCache:increment(matchId, "redScore", 1)

MatchCache.onRemoving:Connect(function(id, data)
    print("Match ended -- red:", data.redScore, "blue:", data.blueScore)
end)

MatchCache:destroy()
```

---

### Reactive client UI

```lua
local Cache = Plinko.define("Players", {
    schema    = { health = 100, coins = 0 },
    replicate = { health = "owner", coins = "owner" },
})

Cache:await():andThen(function(data)
    HealthBar:set(data.health)
    CoinsLabel.Text = data.coins
end)

Cache:watch("health", function(new)
    HealthBar:set(new)
end)

Cache:watch("coins", function(new)
    CoinsLabel.Text = new
end)
```

---

## Contact

| Platform | Handle |
|---|---|
| Roblox | [Kr3ativeKrayon](https://www.roblox.com/users/1911367519/profile) |
| YouTube | [TotallyKr3ative](https://www.youtube.com/channel/UCpNZQoKVclQ74Pk5GmzdQDA) |
| X (Twitter) | [TotallyNotKr3ative](https://x.com/TheRealKr3ative) |
| Email | [TheRealKr3ative@gmail.com](mailto:TheRealKr3ative@gmail.com) |

---

*Last Updated: May 1, 2026*

---
