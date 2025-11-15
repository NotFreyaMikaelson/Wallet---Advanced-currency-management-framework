A production-ready, enterprise-grade currency management framework for Roblox games. Built with performance, security, and scalability in mind.

## Overview

Wallet provides a comprehensive solution for managing multiple currencies in your Roblox game with thread-safe operations, real-time client synchronization, and advanced features like multipliers, exchange rates, and atomic transactions with **full Promise-based API** and **OperationResult** pattern for robust error handling.

## Key Features

### Core Functionality
- **Multi-Currency Support** - Manage unlimited currency types with independent configurations
- **Thread-Safe Operations** - Per-player per-currency Promise-based locking prevents race conditions
- **Real-Time Replication** - Automatic client synchronization via [NetRay](https://devforum.roblox.com/t/netray-high-performance-roblox-networking-library/3592849/1)
- **Full Type Safety** - Complete Luau type annotations with OperationResult pattern
- **Promise-Based API** - All server operations return typed Promises for async handling

### Advanced Systems
- **Atomic Transactions** - Complex multi-currency operations with automatic rollback on failure
- **Temporary Multipliers** - Time-based currency gain modifiers with auto-expiration
- **Exchange Rates** - Built-in currency conversion system with validation
- **Hook System** - Before/after hooks for custom business logic and validation
- **Transaction History** - Automatic logging of the last 50 operations per player
- **Auto-Cap Enforcement** - When setting caps, automatically adjusts all player balances exceeding the limit

### Developer Experience
- **OperationResult Pattern** - Consistent error handling across all operations
- **Promise Chaining** - Compose complex operations with `:andThen()` and `:catch()`
- **Cap & Minimum Constraints** - Automatic enforcement of balance limits
- **Leaderstat Integration** - Support for NumberValue, IntValue, and StringValue types
- **Number Abbreviation** - Built-in formatting (1000 → "1.0k", 1000000 → "1.0M")
- **Comprehensive Validation** - Prevents NaN, Infinity, and unsafe number ranges
- **Data Persistence Ready** - Export/Import system designed for DataStore integration

## Installation

1. Place the `Wallet` ModuleScript in `ReplicatedStorage`
2. Ensure dependencies are installed:
   - [NetRay](https://devforum.roblox.com/t/netray-high-performance-roblox-networking-library/3592849/1) (networking)
   - Janitor (cleanup)
   - Signal (events)
   - Promise (async operations)

## Quick Start

### Server Setup

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Wallet = require(ReplicatedStorage.Wallet.Server)

-- Register currencies with Promise
Wallet.register("Gold", {
    InitialValue = 100,
    ShowOnLeaderstats = true,
    Cap = 999999,
    Minimum = 0,
    Type = "number" -- IntValue in leaderboard
}):andThen(function(result)
    if result.success then
        print("Gold registered!")
    else
        warn("Failed:", result.error)
    end
end)

Wallet.register("Gems", {
    InitialValue = 50,
    ShowOnLeaderstats = true,
    Type = "string" -- StringValue in leaderboard
})

-- Modify balances
Wallet.increase(player, "Gold", 50, "Quest Reward")
    :andThen(function(result)
        if result.success then
            print("New balance:", result.data)
        else
            warn("Failed:", result.error)
        end
    end)

-- Check affordability
Wallet.canAfford(player, {Gold = 100, Gems = 50})
    :andThen(function(result)
        if result.data then
            return Wallet.purchase(player, {Gold = 100, Gems = 50}, "Shop Purchase")
        end
    end)
    :andThen(function(result)
        print("Purchase successful!")
    end)
    :catch(function(err)
        warn("Purchase failed:", err)
    end)

-- Complex transactions with rollback
Wallet.transaction(player,
    {Gold = 100},        -- costs
    {Experience = 500}   -- rewards
):andThen(function(result)
    if result.success then
        print("Transaction completed!")
    end
end)

-- Batch operations
Wallet.batch({
    {player = player1, currency = "Gold", operation = "Increase", amount = 100},
    {player = player2, currency = "Gold", operation = "Increase", amount = 50},
    {player = player3, currency = "Gems", operation = "Set", amount = 200}
}):andThen(function(result)
    print("Processed", #result.data, "operations")
end)
```

### Client Usage

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Wallet = require(ReplicatedStorage.Wallet.Client)

-- Get balances (synchronous)
local gold = Wallet.getBalance("Gold")
local allBalances = Wallet.getAll()

-- Check affordability
if Wallet.canAfford({Gold = 100, Gems = 50}) then
    print("Can afford!")
else
    local missing = Wallet.getMissing({Gold = 100, Gems = 50})
    for currency, amount in missing do
        print("Need", amount, "more", currency)
    end
end

-- Format numbers
print(Wallet.format(1234567, "Gold", true)) -- "1.23M Gold"
print(Wallet.abbreviate(1500)) -- "1.5k"

-- Listen to changes
Wallet.subscribe(function(currency, oldBalance, newBalance)
    print(currency, "changed from", oldBalance, "to", newBalance)
end)

-- Wait for goals with Promises
Wallet.onReached("Gold", 1000, function(amount)
    print("Reached 1000 Gold! Current:", amount)
end)

-- Wait for currency to exist
Wallet.wait("Gold", 5):andThen(function(config)
    if config then
        print("Gold is now available!")
    end
end)
```

## API Reference

### Server API

#### Currency Management

**`register(name: string, config?: CurrencyConfiguration): Promise<OperationResult<boolean>>`**
Register a new currency.

```lua
-- Configuration options:
{
    InitialValue = 0,           -- Starting balance
    ShowOnLeaderstats = false,  -- Display in player list
    Cap = nil,                  -- Maximum balance
    Minimum = nil,              -- Minimum balance
    Type = "string"             -- "string" | "number"
}
```

**`getCurrencies(): Promise<OperationResult<{string}>>`**
Get all registered currency names.

**`getBalance(player: Player, currency: string): Promise<OperationResult<number?>>`**
Get a player's balance for a specific currency.

#### Balance Operations

All balance operations return `Promise<OperationResult<number>>` with the final balance.

**`set(player, currency, amount, source?)`** - Set balance to exact value
**`increase(player, currency, amount, source?)`** - Add to balance (respects multipliers)
**`decrease(player, currency, amount, source?)`** - Subtract from balance
**`multiply(player, currency, amount, source?)`** - Multiply balance
**`divide(player, currency, amount, source?)`** - Divide balance
**`modify(player, currency, operation, amount, source?)`** - Generic operation

```lua
Wallet.increase(player, "Gold", 100, "Daily Reward")
    :andThen(function(result)
        if result.success then
            print("New balance:", result.data)
        else
            warn("Error:", result.error)
        end
    end)
```

#### Transactions

**`canAfford(player, costs): Promise<OperationResult<boolean>>`**
Check if player can afford multiple currencies.

**`purchase(player, costs, source?): Promise<OperationResult<{[string]: number}>>`**
Deduct multiple currencies atomically.

**`transaction(player, costs, rewards, source?): Promise<OperationResult<{costs, rewards}>>`**
Execute complete transaction with automatic rollback on failure.

**`batch(operations): Promise<OperationResult<{{player, currency, success}}>>`**
Process multiple operations in parallel.

```lua
-- Safe purchase with rollback
Wallet.transaction(player,
    {Gold = 500, Gems = 50},
    {Experience = 1000, Points = 100}
):andThen(function(result)
    if result.success then
        givePlayerItem(player, "Legendary Sword")
    end
end):catch(function(err)
    -- Automatically rolled back if any operation fails
    warn("Transaction failed:", err)
end)
```

#### Multipliers & Limits

**`setMultiplier(player, currency, multiplier, duration?): Promise<OperationResult<boolean>>`**
Apply temporary multiplier (only affects Increase operations).

**`getMultiplier(player, currency): Promise<OperationResult<number>>`**
Get current active multiplier.

**`getMultipliers(player): Promise<OperationResult<{[string]: MultiplierData}>>`**
Get all active multipliers for a player.

**`setCap(currency, max): Promise<OperationResult<{adjusted: number}>>`**
Set maximum balance. **Automatically adjusts all players exceeding the cap.**

**`setMinimum(currency, min): Promise<OperationResult<boolean>>`**
Set minimum balance.

**`getCap(currency): Promise<OperationResult<number?>>`**
**`getMinimum(currency): Promise<OperationResult<number?>>`**

```lua
-- Double XP event
Wallet.setMultiplier(player, "Experience", 2, 3600) -- 2x for 1 hour
    :andThen(function(result)
        if result.success then
            print("Double XP activated!")
        end
    end)

-- Set cap and auto-adjust all players
Wallet.setCap("Gold", 100000):andThen(function(result)
    if result.success then
        print("Adjusted", result.data.adjusted, "players")
    end
end)
```

#### Exchange System

**`setExchangeRate(from, to, rate): Promise<OperationResult<boolean>>`**
Set conversion rate between currencies.

**`exchange(player, from, to, amount, source?): Promise<OperationResult<{from, to, rate}>>`**
Convert between currencies with automatic rollback.

```lua
-- Setup exchange rates
Wallet.setExchangeRate("Gold", "Gems", 0.1)  -- 10 Gold = 1 Gem
Wallet.setExchangeRate("Gems", "Gold", 10)   -- 1 Gem = 10 Gold

-- Perform exchange
Wallet.exchange(player, "Gold", "Gems", 1000, "Currency Exchange")
    :andThen(function(result)
        if result.success then
            print("Exchanged", result.data.from, "Gold for", result.data.to, "Gems")
        end
    end)
```

#### Data Persistence

**`export(player): Promise<OperationResult<{[string]: number}>>`**
Export all player balances for DataStore saving.

**`import(player, data): Promise<OperationResult<boolean>>`**
Import balances from saved data.

```lua
-- Save data
local function savePlayerData(player)
    Wallet.export(player):andThen(function(result)
        if result.success then
            local success, err = pcall(function()
                dataStore:SetAsync(player.UserId, {
                    Currencies = result.data,
                    LastSave = os.time()
                })
            end)
            if success then
                print("Saved data for", player.Name)
            end
        end
    end)
end

-- Load data
local function loadPlayerData(player)
    local success, data = pcall(function()
        return dataStore:GetAsync(player.UserId)
    end)
    
    if success and data and data.Currencies then
        Wallet.import(player, data.Currencies):andThen(function(result)
            if result.success then
                print("Loaded data for", player.Name)
            end
        end)
    end
end

Players.PlayerAdded:Connect(loadPlayerData)
Players.PlayerRemoving:Connect(savePlayerData)
```

#### Hooks & Events

**`addMiddleware(callback): () -> ()`**
Add before-change validation hook. Return `false` to block operation.

**`onBalanceChange(callback): () -> ()`**
React to changes after they complete.

**`subscribe(callback)`**
Listen to balance changes (replicated to clients).

```lua
-- Anti-cheat middleware
local removeMiddleware = Wallet.addMiddleware(function(player, currency, oldBal, newBal, source)
    local gain = newBal - oldBal
    if gain > 10000 and source ~= "Admin" then
        warn("Suspicious gain:", player.Name, gain)
        return false -- Block the transaction
    end
    return true
end)

-- Analytics callback
Wallet.onBalanceChange(function(player, currency, oldBal, newBal, source)
    AnalyticsService:TrackCurrencyChange(player, currency, newBal - oldBal, source)
end)
```

#### Utilities

**`giveToAll(currency, amount, excludePlayers?, source?): Promise<OperationResult<{count, results}>>`**
Give currency to all players.

**`getHistory(player, limit?): Promise<OperationResult<[TransactionLog]>>`**
Get transaction history (last 50 operations).

**`abbreviate(amount): string`**
Format large numbers with suffixes.

```lua
-- Server event
Wallet.giveToAll("Gold", 1000, nil, "Server Event"):andThen(function(result)
    if result.success then
        print("Gave gold to", result.data.count, "players")
    end
end)

-- Check player history
Wallet.getHistory(player, 10):andThen(function(result)
    if result.success then
        for _, log in result.data do
            print(log.timestamp, log.operation, log.currency, log.amount)
        end
    end
end)
```

### Client API

#### Balance Queries

**`getBalance(currency): number?`**
Get specific currency balance (synchronous).

**`getAll(): {[string]: number}`**
Get all currency balances.

**`getCurrencies(): {string}`**
Get available currency names.

**`getConfig(currency): CurrencyConfig?`**
Get currency configuration.

#### Affordability

**`canAfford(costs): boolean`**
Check if can afford costs.

**`getMissing(costs): {[string]: number}`**
Calculate missing amounts.

```lua
local cost = {Gold = 500, Gems = 50}

if Wallet.canAfford(cost) then
    -- Request purchase from server
    RemoteEvent:FireServer("Purchase", "Sword")
else
    local missing = Wallet.getMissing(cost)
    for currency, amount in missing do
        print("Need", amount, "more", currency)
    end
end
```

#### Utilities

**`abbreviate(amount): string`**
Format numbers with suffixes.

**`format(amount, currency, abbreviated?): string`**
Format with currency name.

**`wait(currency, timeout?): Promise<CurrencyConfig?>`**
Wait for currency to be registered.

**`waitForBalance(currency, timeout?): Promise<number?>`**
Wait for balance to initialize.

```lua
-- Wait for currency with Promise
Wallet.wait("SeasonalCurrency", 10):andThen(function(config)
    if config then
        print("Seasonal currency loaded!")
        print("Initial value:", config.InitialValue)
    else
        warn("Currency not found")
    end
end)

-- Wait for balance
Wallet.waitForBalance("Gold"):andThen(function(balance)
    print("Gold balance loaded:", balance)
end)
```

#### Events

**`subscribe(callback)`**
Listen to balance changes.

**`onReached(currency, target, callback): () -> ()`**
Trigger when balance reaches target (fires immediately if already reached).

```lua
-- Achievement system
Wallet.onReached("Gold", 10000, function(amount)
    print("Achievement unlocked: Gold Collector!")
    print("Current gold:", amount)
end)

-- UI updates
Wallet.subscribe(function(currency, oldBalance, newBalance)
    if currency == "Gold" then
        updateGoldDisplay(newBalance)
    end
end)
```

## Advanced Examples

### Complete Shop System

```lua
-- Server
local ShopItems = {
    Sword = {Gold = 500, Gems = 50},
    Shield = {Gold = 300},
    Potion = {Gold = 100}
}

local function purchaseItem(player, itemId)
    local cost = ShopItems[itemId]
    if not cost then
        return Promise.reject("Item not found")
    end
    
    return Wallet.canAfford(player, cost)
        :andThen(function(result)
            if not result.data then
                return Promise.reject("Cannot afford")
            end
            return Wallet.purchase(player, cost, "Shop:" .. itemId)
        end)
        :andThen(function(result)
            if result.success then
                giveItemToPlayer(player, itemId)
                return true
            end
        end)
end

-- Client
local function onPurchaseClicked(itemId)
    local cost = ShopItems[itemId]
    
    if Wallet.canAfford(cost) then
        PurchaseRemote:FireServer(itemId)
    else
        local missing = Wallet.getMissing(cost)
        showInsufficientFundsUI(missing)
    end
end
```

### DataStore Integration with Auto-Save

```lua
local DataStoreService = game:GetService("DataStoreService")
local Players = game:GetService("Players")
local playerStore = DataStoreService:GetDataStore("PlayerData")

local AUTO_SAVE_INTERVAL = 300 -- 5 minutes

-- Load player data
local function loadPlayerData(player)
    local success, data = pcall(function()
        return playerStore:GetAsync(player.UserId)
    end)
    
    if success and data then
        if data.Currencies then
            Wallet.import(player, data.Currencies):catch(function(err)
                warn("Failed to import currencies:", err)
            end)
        end
    else
        print("New player or load failed, using defaults")
    end
end

-- Save player data
local function savePlayerData(player)
    return Wallet.export(player):andThen(function(result)
        if result.success then
            local success, err = pcall(function()
                playerStore:SetAsync(player.UserId, {
                    Currencies = result.data,
                    LastSave = os.time()
                })
            end)
            
            if success then
                print("Saved data for", player.Name)
            else
                warn("DataStore error:", err)
            end
        end
    end)
end

-- Auto-save system
local function startAutoSave()
    while true do
        task.wait(AUTO_SAVE_INTERVAL)
        
        for _, player in Players:GetPlayers() do
            task.spawn(savePlayerData, player)
        end
    end
end

Players.PlayerAdded:Connect(loadPlayerData)
Players.PlayerRemoving:Connect(function(player)
    savePlayerData(player):await() -- Wait for save to complete
end)

task.spawn(startAutoSave)

-- Graceful shutdown
game:BindToClose(function()
    local savePromises = {}
    for _, player in Players:GetPlayers() do
        table.insert(savePromises, savePlayerData(player))
    end
    Promise.all(savePromises):await()
end)
```

### Crafting System

```lua
local Recipes = {
    IronSword = {
        costs = {Iron = 10, Wood = 5},
        rewards = {Experience = 50}
    },
    GoldArmor = {
        costs = {Gold = 500, Leather = 20},
        rewards = {Experience = 200}
    }
}

local function craftItem(player, itemId)
    local recipe = Recipes[itemId]
    if not recipe then
        return Promise.reject("Recipe not found")
    end
    
    return Wallet.transaction(player, recipe.costs, recipe.rewards, "Craft:" .. itemId)
        :andThen(function(result)
            if result.success then
                giveItemToPlayer(player, itemId)
                return {
                    success = true,
                    data = itemId,
                    error = nil
                }
            end
        end)
        :catch(function(err)
            return {
                success = false,
                data = nil,
                error = err
            }
        end)
end
```

### Seasonal Event with Multipliers

```lua
local function startSeasonalEvent(duration)
    -- Give all players double currency gain
    for _, player in Players:GetPlayers() do
        Wallet.setMultiplier(player, "Gold", 2, duration)
        Wallet.setMultiplier(player, "Experience", 1.5, duration)
    end
    
    print("Seasonal event started for", duration, "seconds")
    
    task.delay(duration, function()
        print("Seasonal event ended!")
    end)
end

-- Start 2-hour event
startSeasonalEvent(7200)
```

### Advanced Anti-Cheat System

```lua
local SuspiciousGains = {}

Wallet.addMiddleware(function(player, currency, oldBal, newBal, source)
    -- Allow admin sources
    if source and source:match("^Admin:") then
        return true
    end
    
    local gain = newBal - oldBal
    
    -- Track suspicious gains
    if gain > 5000 then
        SuspiciousGains[player] = (SuspiciousGains[player] or 0) + 1
        
        if SuspiciousGains[player] > 3 then
            warn("EXPLOIT DETECTED:", player.Name, "suspicious gains:", SuspiciousGains[player])
            player:Kick("Suspicious activity detected")
            return false
        end
        
        warn("Suspicious gain:", player.Name, gain, currency)
    end
    
    -- Validate source
    local validSources = {"Quest", "Shop", "Daily", "Achievement"}
    if source and not table.find(validSources, source:match("^([^:]+)")) then
        warn("Invalid source:", player.Name, source)
        return false
    end
    
    return true
end)

-- Analytics
Wallet.onBalanceChange(function(player, currency, oldBal, newBal, source)
    local change = newBal - oldBal
    AnalyticsService:RecordEvent("CurrencyChange", {
        Player = player.UserId,
        Currency = currency,
        Amount = change,
        Source = source,
        Timestamp = os.time()
    })
end)
```

## Best Practices

1. **Always use Promises** - Chain operations with `:andThen()` and handle errors with `:catch()`
2. **Check `result.success`** - Always validate OperationResult before using `result.data`
3. **Use `canAfford()` before purchases** - Provide better UX with missing funds display
4. **Use `transaction()` for complex operations** - Ensures atomic rollback on failure
5. **Add middleware for validation** - Implement business logic and anti-cheat in hooks
6. **Use source parameter** - Track where currency changes originate for analytics
7. **Set caps on farmable currencies** - Prevent integer overflow and exploits
8. **Export/Import for DataStores** - Use built-in methods for clean persistence
9. **Handle Promise rejections** - Always add `.catch()` to prevent silent failures
10. **Use batch operations** - Process multiple players efficiently

## Performance

- **Promise-Based Async** - Non-blocking operations with efficient concurrency
- **Optimized Replication** - Only syncs changes via NetRay, not entire state
- **Memory Efficient** - Rolling transaction history (last 50 entries per player)
- **Lock Granularity** - Per-player per-currency Promise locks minimize blocking
- **Automatic Cleanup** - Janitor-based resource management prevents memory leaks

## Security Features

- **Thread-Safe Locking** - Promise-based locks prevent race conditions
- **Number Validation** - Rejects NaN, Infinity, and unsafe integers (>9e15)
- **Balance Constraints** - Automatic enforcement of caps and minimums
- **Hook Validation** - Before-hooks can reject suspicious transactions
- **Transaction Logging** - Audit trail with source tracking
- **Type Safety** - Full Luau annotations prevent runtime errors
## License

MIT License - Free for commercial and personal use

---

**For issues, questions, or feature requests, please contact the framework maintainer.**

### Special Thanks
- [NetRay](https://devforum.roblox.com/t/netray-high-performance-roblox-networking-library/3592849/1) - High-performance networking library
