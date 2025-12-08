# Admin Commands

Server management commands for 51alpha systems.

## Access Levels

| Level | Commands |
|-------|----------|
| Player | Basic info queries |
| Counselor | Player assistance |
| GameMaster | Event management |
| Seer | Content testing |
| Administrator | Full system control |

---

## Core Commands

### Configuration

| Command | Level | Description |
|---------|-------|-------------|
| `[s51a reload config]` | Admin | Reload configuration |
| `[s51a status]` | GM | Show system status |
| `[s51a version]` | Player | Show version info |

### Database

| Command | Level | Description |
|---------|-------|-------------|
| `[s51a db status]` | Admin | Database connection status |
| `[s51a db flush]` | Admin | Flush pending operations |

---

## Faction Commands

### Player Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[faction]` | Player | Open faction status gump |
| `[factionstatus]` | Player | Show faction info |

### GM Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[faction set <guild> <faction>]` | GM | Set guild faction |
| `[faction clear <guild>]` | GM | Remove guild from faction |
| `[faction cooldown <guild> clear]` | GM | Clear change cooldown |

### Admin Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[faction points <player> <amount>]` | Admin | Add/remove faction points |
| `[faction reset <player>]` | Admin | Reset player faction data |

---

## Siege Commands

### Player Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[siege]` | Player | Open siege status gump |

### GM Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[siege start <city>]` | GM | Force start siege |
| `[siege stop]` | GM | End active siege |
| `[siege status]` | GM | Show siege state |
| `[siege score <faction> <points>]` | GM | Adjust score |

### Admin Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[siege toggles]` | Admin | Show objective toggles |
| `[siege sigil <on/off>]` | Admin | Enable/disable sigil |
| `[siege altars <on/off>]` | Admin | Enable/disable altars |
| `[siege config]` | Admin | Show siege configuration |

---

## Tournament Commands

### Player Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[tournament]` | Player | Open tournament gump |
| `[tournament register]` | Player | Register for next tournament |
| `[tournament withdraw]` | Player | Withdraw registration |

### GM Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[tournament start]` | GM | Force start tournament |
| `[tournament cancel]` | GM | Cancel active tournament |
| `[tournament status]` | GM | Show tournament state |
| `[tournament advance <player>]` | GM | Force advance player |

### Admin Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[tournament coins <player> <amount>]` | Admin | Add/remove coins |
| `[tournament reset]` | Admin | Reset tournament system |

---

## Daily Content Commands

### Player Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[bounty]` | Player | Open bounty board |
| `[quest]` | Player | Show faction quest |

### GM Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[bounty reset]` | GM | Force daily reset |
| `[bounty complete <player> <task>]` | GM | Complete task for player |
| `[quest spawn]` | GM | Force boss spawn |
| `[quest kill]` | GM | Kill active boss |

### Admin Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[bounty generate]` | Admin | Generate new bounties |
| `[quest location <id>]` | Admin | Set quest location |

---

## Currency Commands

### GM Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[silver <player>]` | GM | Show player silver |
| `[silver add <player> <amount>]` | GM | Add silver |
| `[silver remove <player> <amount>]` | GM | Remove silver |

### Admin Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[silver audit <player>]` | Admin | Show transaction log |
| `[factionpoints audit <player>]` | Admin | Show FP log |

---

## Glicko Commands

### Player Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[rating]` | Player | Show own rating |
| `[leaderboard]` | Player | Open leaderboard gump |

### GM Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[rating <player>]` | GM | Show player rating |
| `[rating history <player>]` | GM | Show match history |

### Admin Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[rating set <player> <rating>]` | Admin | Set player rating |
| `[rating reset <player>]` | Admin | Reset to default |
| `[rating period force]` | Admin | Force rating period end |

---

## NPE Commands

### Player Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[renounceyoung]` | Player | Renounce young status |

### GM Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[young <player>]` | GM | Check young status |
| `[young extend <player> <days>]` | GM | Extend protection |
| `[young end <player>]` | GM | End protection early |

### Admin Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[young grant <player>]` | Admin | Grant young status |
| `[quest progress <player>]` | Admin | Show quest progress |
| `[quest complete <player> <quest>]` | Admin | Complete quest |

---

## Talisman Commands

### GM Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[talisman <player>]` | GM | Show talisman state |
| `[talisman disable <player>]` | GM | Force PvP disable |
| `[talisman enable <player>]` | GM | Clear disable timer |

---

## Economy Commands

### GM Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[rental <building>]` | GM | Show rental status |
| `[rental evict <building>]` | GM | Force eviction |

### Admin Commands

| Command | Level | Description |
|---------|-------|-------------|
| `[economy stats]` | Admin | Show economy dashboard |
| `[economy audit]` | Admin | Run integrity check |
| `[goldtrack <player>]` | Admin | Track gold flow |

---

## Maintenance Commands

### Admin Only

| Command | Level | Description |
|---------|-------|-------------|
| `[RemoveAllFC]` | Admin | Strip FC from all items |
| `[s51a backup]` | Admin | Trigger database backup |
| `[s51a maintenance <on/off>]` | Admin | Toggle maintenance mode |

---

## Debug Commands

### Seer+

| Command | Level | Description |
|---------|-------|-------------|
| `[s51a debug combat]` | Seer | Combat debug output |
| `[s51a debug siege]` | Seer | Siege debug output |
| `[s51a debug talisman]` | Seer | Talisman debug output |

### Admin Only

| Command | Level | Description |
|---------|-------|-------------|
| `[s51a debug all]` | Admin | Full debug mode |
| `[s51a trace <player>]` | Admin | Trace player actions |
| `[s51a perf]` | Admin | Performance metrics |

---

## Command Examples

### Force Start Siege
```
[siege start jhelom]
> Siege started in Jhelom. Duration: 30 minutes.
```

### Adjust Faction Points
```
[faction points PlayerName 500]
> Added 500 faction points to PlayerName. New balance: 1500.
```

### Check Tournament Status
```
[tournament status]
> Tournament #15: In Progress
> Round: 2 of 4
> Remaining: 8 players
> Next match: Arena 3 in 45 seconds
```

### Remove FC From World
```
[RemoveAllFC]
> Removed Faster Casting from 1,247 items.
```

### Rating Period Force
```
[rating period force]
> Rating period ended. 156 players updated.
```

---

## Logging

All admin commands are logged to:
- Console: `[51alpha] Admin: <command> by <player>`
- Database: `s51a_events` table with `event_type = 'admin_command'`

```sql
SELECT * FROM s51a_events 
WHERE event_type = 'admin_command' 
ORDER BY created_at DESC 
LIMIT 50;
```
