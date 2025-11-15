# Wallet: Advanced-currency management framework
A production-ready, enterprise-grade currency management framework for Roblox games. Built with performance, security, and scalability in mind.

## Overview

Wallet provides a comprehensive solution for managing multiple currencies in your Roblox game with thread-safe operations, real-time client synchronization, and advanced features like multipliers, rate limiting, and atomic transactions.

## Key Features
### Core Functionality
- **Multi-Currency Support** - Manage unlimited currency types with independent configurations
- **Thread-Safe Operations** - Per-player per-currency locking prevents race conditions
- **Real-Time Replication** - Automatic client synchronization via [NetRay](https://devforum.roblox.com/t/netray-high-performance-roblox-networking-library/3592849/1)
- **Type Safety** - Full Luau type annotations for IntelliSense support
### Advanced Systems
- **Atomic Transactions** - Complex multi-currency operations with automatic rollback on failure
- **Rate Limiting** - Sliding window algorithm to prevent abuse and exploits
- **Temporary Multipliers** - Time-based currency gain modifiers with auto-expiration
- **Exchange Rates** - Built-in currency conversion system with validation
- **Hook System** - Before/after hooks for custom business logic and validation
- **Transaction History** - Automatic logging of the last 100 operations per player
### Developer Experience
- **Promise-Based API** - Async operations with `:andThen()` and `:catch()` chaining
- **Cap & Minimum Constraints** - Automatic enforcement of balance limits
- **Leaderboard Integration** - Optional display in player list
- **Number Abbreviation** - Built-in formatting (1000 → "1.0K", 1000000 → "1.0M")
- **Comprehensive Validation** - Prevents NaN, Infinity, and unsafe number ranges
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
local Wallet = require(ReplicatedStorage.Wallet)
-- Register currencies
Wallet.registerCurrency("Gold", 100, true, true)
Wallet.registerCurrency("Gems", 50, true, true)
Wallet.registerCurrency("Experience", 0, true, false)

-- Set constraints
Wallet.setCap("Gold", 999999)
Wallet.setMinimum("Gold", 0)

-- Configure rate limiting
Wallet.setRateLimit("Gold", 10) -- 10 operations per minute

-- Modify balances
Wallet.modifyBalance(player, "Gold", "Increase", 50)
Wallet.modifyBalance(player, "Gems", "Set", 100)

-- Check affordability
if Wallet.canAfford(player, {Gold = 100, Gems = 50}) then
    Wallet.deductMultiple(player, {Gold = 100, Gems = 50})
        :andThen(function()
            print("Purchase successful!")
        end)
end

-- Complex transactions
Wallet.transaction(player,
    {Gold = 100},           -- costs
    {Experience = 500}      -- rewards
):andThen(function(result)
    print("Transaction completed!")
end)
```
### Client Usage
```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Wallet = require(ReplicatedStorage.Wallet)

-- Get balances
local gold = Wallet.getBalance("Gold")
local allBalances = Wallet.getAllBalances()

-- Check affordability
if Wallet.canAfford({Gold = 100, Gems = 50}) then
    print("Can afford!")
else
    local missing = Wallet.getMissingFunds({Gold = 100, Gems = 50})
    for currency, amount in missing do
        print("Need", amount, "more", currency)
    end
end

-- Format numbers
print(Wallet.formatCurrency(1234567, "Gold")) -- "1.2M Gold"

-- Listen to changes
Wallet.subscribeToBalanceChanged(function(currency, oldBalance, newBalance)
    print(currency, "changed from", oldBalance, "to", newBalance)
end)

-- Wait for goals
Wallet.onBalanceReached("Gold", 1000, function(amount)
    print("Reached 1000 Gold!")
end)
```
## API Reference

### Server API

#### Currency Management
- `registerCurrency(name, initialValue?, syncToJoin?, showOnLeaderstats?)` - Register new currency
- `unregisterCurrency(name)` - Remove currency from system
- `getCurrencies(excludeList?)` - Get all registered currencies

#### Balance Operations
- `getBalance(player, currency)` - Get current balance
- `modifyBalance(player, currency, operation, amount)` - Modify balance synchronously
- `modifyBalanceAsync(player, currency, operation, amount)` - Modify balance with Promise
- `canAfford(player, costs)` - Check if player can afford multiple currencies

#### Transactions
- `deductMultiple(player, costs)` - Deduct multiple currencies atomically
- `transaction(player, costs, rewards)` - Execute complete transaction with rollback
- `batchModify(operations)` - Process multiple operations in batch

#### Multipliers & Limits
- `setMultiplier(player, currency, multiplier, duration?)` - Apply temporary multiplier
- `getMultiplier(player, currency)` - Get current multiplier
- `clearMultipliers(player)` - Remove all multipliers
- `setCap(currency, max)` - Set maximum balance
- `setMinimum(currency, min)` - Set minimum balance
- `setRateLimit(currency, maxOpsPerMinute)` - Configure rate limiting

#### Exchange System
- `setExchangeRate(from, to, rate)` - Set conversion rate
- `exchange(player, from, to, amount)` - Convert between currencies

#### Hooks & Events
- `onBeforeBalanceChange(callback)` - Validate changes before they occur
- `onAfterBalanceChange(callback)` - React to changes after they complete
- `subscribeToBalanceChanged(callback)` - Listen to balance changes
- `subscribeToCurrencyRegistered(callback)` - Listen to currency registration
- `subscribeToCurrencyUnregistered(callback)` - Listen to currency removal

#### Utilities
- `getTransactionHistory(player, limit?)` - Get transaction log
- `abbreviateNumber(amount)` - Format large numbers
- `giveToAll(currency, amount, excludePlayers?)` - Give currency to all players

### Client API

#### Balance Queries
- `getBalance(currency)` - Get specific currency balance
- `getAllBalances()` - Get all currency balances
- `getCurrencies(excludeList?)` - Get available currencies
- `getCurrencyConfig(currency)` - Get currency configuration

#### Affordability
- `canAfford(costs)` - Check if can afford costs
- `getMissingFunds(costs)` - Calculate missing amounts

#### Utilities
- `abbreviateNumber(amount)` - Format numbers
- `formatCurrency(amount, currency)` - Format with currency name
- `waitForCurrency(currency, timeout?)` - Wait for currency to exist
- `waitForBalance(currency, timeout?)` - Wait for balance to initialize

#### Events
- `subscribeToBalanceChanged(callback)` - Listen to balance changes
- `subscribeToCurrencyRegistered(callback)` - Listen to new currencies
- `subscribeToCurrencyUnregistered(callback)` - Listen to removed currencies
- `onBalanceReached(currency, target, callback)` - Trigger when goal reached

## Security Features

- **Thread-Safe Locking** - Prevents concurrent modification race conditions
- **Rate Limiting** - Sliding window algorithm prevents spam and exploits
- **Number Validation** - Rejects NaN, Infinity, and unsafe integers
- **Balance Constraints** - Automatic enforcement of caps and minimums
- **Hook Validation** - Before-hooks can reject suspicious transactions
- **Transaction Logging** - Audit trail for anti-cheat systems

## Performance

- **Optimized Replication** - Only syncs changes, not entire state
- **Memory Efficient** - Rolling transaction history (last 100 entries)
- **Lock Granularity** - Per-player per-currency locking for minimal blocking
- **Async Operations** - Non-blocking Promise-based API
- **Automatic Cleanup** - Janitor-based resource management

## Use Cases

- **Economy Systems** - Coins, gems, premium currency
- **Progression** - Experience points, skill points
- **Battle Passes** - Seasonal currencies with caps
- **Trading Systems** - Exchange rates between currencies
- **Reward Systems** - Daily bonuses with multipliers
- **Shop Systems** - Multi-currency purchases
- **Crafting** - Resource conversion and exchange

## Advanced Examples

### Purchase System with Rollback
```lua
local function purchaseItem(player, itemId)
    local itemCost = {Gold = 100, Gems = 50}
    local itemRewards = {Experience = 500}
    
    return Wallet.transaction(player, itemCost, itemRewards)
        :andThen(function()
            giveItemToPlayer(player, itemId)
            return true
        end)
        :catch(function(err)
            warn("Purchase failed:", err)
            return false
        end)
end
```

### Double XP Event
```lua
local function startDoubleXPEvent(duration)
    for _, player in Players:GetPlayers() do
        Wallet.setMultiplier(player, "Experience", 2, duration)
    end
end
```

### Anti-Cheat Hook
```lua
Wallet.onBeforeBalanceChange(function(player, currency, oldBal, newBal)
    local gain = newBal - oldBal
    if gain > 10000 then
        warn("Suspicious gain detected:", player.Name, gain)
        return false -- Block the transaction
    end
    return true
end)
```

### Currency Exchange Shop
```lua
Wallet.setExchangeRate("Gold", "Gems", 0.1) -- 10 Gold = 1 Gem
Wallet.setExchangeRate("Gems", "Gold", 10)  -- 1 Gem = 10 Gold

Wallet.exchange(player, "Gold", "Gems", 1000)
    :andThen(function(result)
        print("Exchanged", result.from, "Gold for", result.to, "Gems")
    end)
```

## Best Practices

1. **Always use `canAfford()` before deductions** to provide better UX
2. **Use `transaction()` for complex operations** to ensure atomic rollback
4. **Set rate limits on farmable currencies** to prevent exploits
5. **Use hooks for business logic** instead of modifying core system
6. **Log important transactions** for analytics and debugging
7. **Use async operations** for non-critical paths to avoid blocking
8. **Set caps on premium currencies** to prevent integer overflow exploits
> ## License
> MIT License - Free for commercial and personal use

**For issues, questions, or feature requests, please refer to the documentation or contact the framework maintainer.**
### Special thanks to [NetRay](https://devforum.roblox.com/t/netray-high-performance-roblox-networking-library/3592849/1) (High perfomance Networking library)
### Creator Store: [Wallet](https://create.roblox.com/store/asset/123585952224629/Wallet) -- Not actually working

**Version:** 1.0.0
**Maintained by:** Cerso98zt (AlwaysAndForever)

> # **Version 1.0.0**
> - Initial stable release of **Wallet**
> - All core systems fully implemented and production-tested
> - Includes multi-currency management, atomic transactions, multipliers, rate limiting, exchange system, hooks, and real-time synchronization
> - API marked as stable
> - Ready for enterprise-scale deployments
> production.
