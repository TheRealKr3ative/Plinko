# Plinko -- Live Registry Library

[![Static Badge](https://img.shields.io/badge/build-v0.0.0--beta--(unreleased)-black)](https://github.com/TheRealKr3ative)
![Static Badge](https://img.shields.io/badge/stability-stable-green)

A live in-memory cache for Roblox player and standalone data. Define a schema, write to it, and Plinko handles dirty tracking, replication, and persistence hooks -- without touching DataStore directly.

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

- **Schema-first** -- declare field names and defaults upfront; every write is validated against the schema
- **Dirty tracking** -- only changed fields are passed to `onSave` -- unchanged keys never trigger a flush
- **Thenable hooks** -- `onLoad` and `onSave` return Promises -- you own the persistence logic
- **Replication** -- per-key visibility scoped to `"owner"` or `"all"` via Schema under the hood
- **Middleware** -- intercept and transform values before they land on a field
- **Snapshots** -- named point-in-time copies with diff and rollback support
- **Standalone caches** -- key by anything, not just players
- **Multiple caches** -- `Plinko.define()` returns isolated instances that coexist independently

---

## Installation

Place Plinko and its dependencies inside `ReplicatedStorage`:

```
ReplicatedStorage/
    Plinko/
        Plinko.lua
        GoodSignal.lua
        Promise.lua
        Schema.lua      -- optional, required for replication
```

Require from both server and client:

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

`Plinko.define()` returns a **Cache** -- an isolated live store with its own schema, replication config, save hooks, and signals. Multiple caches coexist without interfering with each other.

### Schema

The `schema` table defines field names and their default values. Defaults are used for type validation on every write and serve as the baseline for `:reset()` and `:migrate()`. Writing a key that doesn't exist in the schema is rejected.

### Dirty Tracking

Every `:set()` or `:patch()` marks affected fields dirty. When a flush runs -- either on the auto-save interval, on `PlayerRemoving`, or manually via `:flush()` -- only those dirty fields are passed to `onSave`. Keys that were never written, or written with the same value they already held, are excluded.

### Thenables

`onLoad` and `onSave` must return a Promise. Plinko chains off whatever you return -- letting you plug in any persistence layer without the library dictating how you talk to DataStore.

---

## API Reference

### `Plinko.define(name, config)`

Defines a new cache and returns it. Calling `define` with the same name twice returns the existing instance.

```lua
local Cache = Plinko.define("Players", {
    schema = { health = 100, coins = 0 },

    replicate = { health = "owner", coins = "owner" },

    visibility = "public",

    middleware = {
        health = function(value, old)
            return math.clamp(value, 0, 100)
        end,
    },

    save   = { interval = 60, onLeave = true },
    onLoad = function(player) return Promise.resolve({}) end,
    onSave = function(player, data) return Promise.resolve() end,
})
```

| Field | Type | Description |
|---|---|---|
| `schema` | `table` | Field names and default values |
| `replicate` | `table?` | Per-key replication scope -- `"owner"` or `"all"` |
| `visibility` | `string?` | `"public"` (default) or `"private"` |
| `middleware` | `table?` | Per-key transform functions that run before writes land |
| `save.interval` | `number?` | Auto-flush interval in seconds -- `0` disables |
| `save.onLeave` | `boolean?` | Flush on `PlayerRemoving`. Defaults to `true` |
| `onLoad` | `function?` | Called on player join -- must return a Promise resolving to a data table |
| `onSave` | `function?` | Called on flush -- receives `(player, dirtyFields)`, must return a Promise |
| `standalone` | `boolean?` | Detaches from the player lifecycle -- see [Standalone Caches](#standalone-caches) |

---

### Reading & Writing

```lua
Cache:get(player, key)            -- read one field
Cache:set(player, key, value)     -- write a field -- runs middleware, validates type, marks dirty
Cache:increment(player, key, n?)  -- numeric shorthand -- n defaults to 1
Cache:patch(player, delta)        -- merge a partial table -- runs middleware per field
Cache:reset(player, key)          -- revert a field to its schema default
Cache:flush(player)               -- trigger a manual save -- returns a thenable
Cache:loaded(player)              -- returns true once onLoad has resolved
Cache:await(player?)              -- thenable resolving to the loaded data table -- omit player on the client
Cache:lock(player)                -- freeze all writes
Cache:unlock(player)              -- resume writes after a lock
```

---

### Visibility

```lua
Cache:hide(player)    -- make this player's cache invisible to other clients
Cache:show(player)    -- revert to the visibility set in the definition
```

---

### Querying

```lua
Cache:find(function(player, data)
    return data.rank == "Gold"
end)
-- returns { [Player]: data } for every entry that matches

Cache:some(function(player, data)
    return data.health <= 0
end)
-- returns true if at least one entry matches

Cache:count()                          -- total number of loaded entries
Cache:count(function(player, data)     -- count entries matching a predicate
    return data.kills > 10
end)
```

---

### Bulk Operations

```lua
Cache:setAll(key, value)    -- write a field for every loaded entry
Cache:patchAll(delta)       -- merge a partial table into every loaded entry
Cache:flushAll()            -- flush all dirty entries -- returns a thenable
Cache:resetAll(key)         -- revert a field to its schema default for every loaded entry
```

---

### Dirty Inspection

```lua
Cache:dirty(player)           -- returns { [key]: newValue } for every unsaved change
Cache:isDirty(player, key)    -- returns true if a specific field is dirty
Cache:clearDirty(player)      -- discard dirty flags without saving
```

---

### Events & Watching

```lua
Cache:watch(player, key, function(new, old) end)      -- fires when a specific field changes
Cache:onChange(player, function(key, new, old) end)   -- fires on any change to a player's data
Cache:onFlushFor(player, function(dirty) end)         -- fires after each flush for a player

Cache.onAdded:Connect(function(player, data) end)     -- fires when a player's data loads
Cache.onRemoving:Connect(function(player, data) end)  -- fires just before data is released
Cache.onFlush:Connect(function(player, dirty) end)    -- fires after any flush
Cache.onError:Connect(function(player, err) end)      -- fires when onLoad or onSave rejects
```

> On the client, `:watch()` and `:onChange()` omit the player argument -- they always refer to the local player's data.

---

### Destruction

```lua
Cache:destroy()
-- flushAll -> release all entries -> fire onRemoving -> destroy all signals
```

---

## Replication

Add a `replicate` table to the definition to control which fields reach clients and who can see them.

```lua
replicate = {
    health = "owner",    -- only the owning client receives updates to this field
    rank   = "all",      -- all clients receive updates to this field
}
```

Replication fires automatically after every `:set()` and `:patch()` call for flagged fields. To push changes manually:

```lua
Cache:push(player, key)    -- replicate one field immediately
Cache:pushAll(player)      -- replicate all flagged fields immediately
```

Setting `visibility = "private"` in the definition prevents other clients from receiving `"all"` fields for a given player. Override this per player at runtime with `:hide()` and `:show()`.

Replication requires Schema. If the `replicate` field is omitted from the definition entirely, Schema is never loaded or used.

---

## Snapshots

Snapshots are named, point-in-time copies of a player's current data. They're useful for pre-round baselines, undo flows, or comparing state after a sequence of mutations.

```lua
Cache:snapshot(player, "beforeCombat")          -- capture current state
Cache:getSnapshot(player, "beforeCombat")       -- read a snapshot without modifying state
Cache:diff(player, "beforeCombat")              -- returns { [key]: { old, new } } vs current state
Cache:migrate(player, "beforeCombat")           -- overwrite current state with snapshot, marks fields dirty
Cache:destroySnapshot(player, "beforeCombat")   -- remove one snapshot
Cache:destroySnapshots(player)                  -- remove all snapshots for a player
```

Snapshot data is also accessible directly when needed:

```lua
Cache._snapshots[player]["beforeCombat"]    -- raw data table
```

---

## Standalone Caches

Set `standalone = true` to decouple a cache from the player lifecycle entirely. The cache won't listen to `PlayerAdded` or `PlayerRemoving`, and it doesn't support `onLoad` or `onSave`. Lifecycle is fully manual -- useful for match state, zone data, or any server-side store keyed by something other than a player.

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

### Clamping health with middleware

```lua
local Cache = Plinko.define("Players", {
    schema = { health = 100 },
    middleware = {
        health = function(value)
            return math.clamp(value, 0, 100)
        end,
    },
})

Cache:set(player, "health", 9999)
Cache:get(player, "health")    -- 100
```

---

### Combat snapshot and rollback

```lua
Cache:snapshot(player, "preRound")

Cache:set(player, "health", 20)
Cache:increment(player, "deaths", 1)

local changes = Cache:diff(player, "preRound")

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

*Last Updated: May 2, 2026*
