# Portals TDM Leaderboard System

A complete Team Deathmatch HUD system built with iframes for Portals.

## Repository

**GitHub:** https://github.com/jbgibbons093/portals-tdm-leaderboard
**GitHub Pages:** https://jbgibbons093.github.io/portals-tdm-leaderboard/

## Files

| File | Purpose | URL |
|------|---------|-----|
| `index.html` | In-game HUD (scores + timer) | `/index.html?v=X` |
| `team-select.html` | Pre-game lobby/matchmaking | `/team-select.html?v=X` |
| `game-over.html` | Post-game results screen | `/game-over.html?winner=red&v=X` |

## Design

All three iframes use a **Halo-inspired military sci-fi aesthetic**:
- Cyan (#00d4ff) primary accent color
- Angular shapes using CSS `clip-path`
- Animated scan lines
- Corner bracket decorations
- Orbitron font for headers
- Share Tech Mono font for scores/data
- Dark backgrounds with subtle grid patterns

## Message Formats

### Portals → Iframe

| Message | Format | Example | Used By |
|---------|--------|---------|---------|
| **Game start** | `game_start` | `game_start` | HUD - starts local timer |
| Red score only | `red_SCORE` | `red_25` | HUD |
| Blue score only | `blue_SCORE` | `blue_18` | HUD |
| Reset | `reset` | `reset` | HUD - stops timer, resets display |
| Player slot | `my_slot_X` | `my_slot_3` | Team-select |
| Lobby sync | `lobbysync_C1_T1_R1_C2_T2_R2...` | 8 slots × 3 values | Team-select |
| Countdown | `countdown_X` | `countdown_3` | Team-select |
| Scores (game-over) | `scores_RED_BLUE` | `scores_50_32` | Game-over |
| Sync all data (legacy) | `sync_TIME_RED_BLUE` | `sync_120_25_18` | HUD (backwards compatible) |
| Time only (legacy) | `time_ELAPSED` | `time_120` | HUD (ignored - uses local timer) |

### Iframe → Portals

| Task | When Sent | Purpose |
|------|-----------|---------|
| `team_select_ready` | On team-select load | Signals iframe ready for slot assignment |
| `gameover_ready` | On game-over load | Signals ready for score data |
| `join_red` | Button click | Player joins red team |
| `join_blue` | Button click | Player joins blue team |
| `player_ready` | Button click | Player ready to start |
| `player_unready` | Button click | Player cancels ready |
| `all_players_ready` | Auto | All players ready, start countdown |
| `clear_all_slots` | Reset Lobby button | Clears all lobby slots (admin function) |
| `reset_my_lobby` | Auto (on external clear) | Resets local My_Slot and My_Team_Local |
| `timer_red_wins` | Auto | Red reached 50 or time ran out with lead |
| `timer_blue_wins` | Auto | Blue reached 50 or time ran out with lead |
| `leaderboard_ready` | On HUD load | HUD initialized |

## Portals Setup Requirements

### Variables (Multiplayer)
- `Red_Score` - Red team points
- `Blue_Score` - Blue team points
- `Slot1_Name` through `Slot8_Name` - Lobby slot claimed (0=empty, 1=claimed)
- `Slot1_Team` through `Slot8_Team` - Team assignments (0=none, 1=red, 2=blue)
- `Slot1_Ready` through `Slot8_Ready` - Ready status (0=not ready, 1=ready)

**Note:** Timer is handled locally by the leaderboard iframe. No server-side timer variables needed.

### Variables (Single Player)
- `Player_Team` - This player's team (0=none, 1=red, 2=blue)
- `My_Slot` - This player's lobby slot (1-8)
- `My_Team_Local` - Local team choice for lobby

### Key Tasks

**start_game**
- Trigger: Via iframe (all players ready)
- Action: Teleport players to spawns, send `game_start` to iframes
- **Important:** Must include `Send Message To Iframes: game_start` effect

**team_select_ready**
- Trigger: Iframe Message Received
- On Completed: Trigger `claim_slot` to assign player their slot
- Reset: Set back to NotActive when player enters lobby

**UpdateRedScore / UpdateBlueScore**
- Trigger: Value Updated on Red_Score / Blue_Score
- Action: Send `red_|Red_Score|` or `blue_|Blue_Score|` to iframes
- Ensures non-scoring players see score updates

## Timing Patterns

### First-Load Ready Handshake
```
Player enters lobby → Reset team_select_ready to NotActive → Open iframe
        ↓
Iframe loads → 300ms delay → Sends team_select_ready (Completed)
        ↓
team_select_ready triggers claim_slot → Sends my_slot_X to iframe
        ↓
Iframe shows matchmaking UI
```

### Game In Progress Detection
The team-select iframe detects active games by watching for `sync_` messages. If received, it shows "MATCH IN PROGRESS" and blocks matchmaking.

### Game Over Handshake
```
Win condition triggers → Open game-over.html?winner=red
        ↓
Iframe loads → Reads winner from URL → Sends gameover_ready
        ↓
gameover_ready triggers → Send scores_RED_BLUE
        ↓
Iframe displays full results
```

## Event-Driven Lobby System (v37)

The team-select iframe uses an **event-driven** architecture instead of polling. Updates are pushed instantly when any player action occurs.

### How It Works

1. Player action (join team, ready up, etc.) triggers a task
2. Task updates variables AND triggers `broadcast_lobby`
3. `broadcast_lobby` (Multiplayer) sends `lobbysync_...` to ALL players' iframes
4. All iframes update instantly

### broadcast_lobby Task

| Component | Configuration |
|-----------|---------------|
| **Task Name** | `broadcast_lobby` |
| **Type** | Multiplayer |
| **Initial State** | NotActive |

**Effect 1 - Send sync to all iframes:**
(Send Message To Iframes effect)
```
lobbysync_|Slot1_Name|_|Slot1_Team|_|Slot1_Ready|_|Slot2_Name|_|Slot2_Team|_|Slot2_Ready|_|Slot3_Name|_|Slot3_Team|_|Slot3_Ready|_|Slot4_Name|_|Slot4_Team|_|Slot4_Ready|_|Slot5_Name|_|Slot5_Team|_|Slot5_Ready|_|Slot6_Name|_|Slot6_Team|_|Slot6_Ready|_|Slot7_Name|_|Slot7_Team|_|Slot7_Ready|_|Slot8_Name|_|Slot8_Team|_|Slot8_Ready|
```

**Effect 2 - Reset:**
```
SetTask('broadcast_lobby', 'NotActive', 0.1)
```

### Tasks That Trigger broadcast_lobby

These tasks should include `SetTask('broadcast_lobby', 'Active', 0.1)` as an effect:
- `claim_slot` - after assigning slot
- `join_red` - after setting team
- `join_blue` - after setting team
- `player_ready` - after setting ready
- `player_unready` - after clearing ready
- `clear_my_slot` - after clearing slot
- `clear_all_slots` - after clearing all slots (use 0.5 delay for variable sync)

### Reset Lobby Feature

The team-select iframe has a "Reset Lobby" button that clears all slots. Useful for clearing stale slots from disconnected players.

**clear_all_slots Task (Multiplayer):**

**Effect 1 - Clear all slots:**
```
if(SetVariable('Slot1_Name', 0.0, 0.0) == 0, if(SetVariable('Slot1_Team', 0.0, 0.0) == 0, if(SetVariable('Slot1_Ready', 0.0, 0.0) == 0, if(SetVariable('Slot2_Name', 0.0, 0.0) == 0, if(SetVariable('Slot2_Team', 0.0, 0.0) == 0, if(SetVariable('Slot2_Ready', 0.0, 0.0) == 0, if(SetVariable('Slot3_Name', 0.0, 0.0) == 0, if(SetVariable('Slot3_Team', 0.0, 0.0) == 0, if(SetVariable('Slot3_Ready', 0.0, 0.0) == 0, if(SetVariable('Slot4_Name', 0.0, 0.0) == 0, if(SetVariable('Slot4_Team', 0.0, 0.0) == 0, if(SetVariable('Slot4_Ready', 0.0, 0.0) == 0, if(SetVariable('Slot5_Name', 0.0, 0.0) == 0, if(SetVariable('Slot5_Team', 0.0, 0.0) == 0, if(SetVariable('Slot5_Ready', 0.0, 0.0) == 0, if(SetVariable('Slot6_Name', 0.0, 0.0) == 0, if(SetVariable('Slot6_Team', 0.0, 0.0) == 0, if(SetVariable('Slot6_Ready', 0.0, 0.0) == 0, if(SetVariable('Slot7_Name', 0.0, 0.0) == 0, if(SetVariable('Slot7_Team', 0.0, 0.0) == 0, if(SetVariable('Slot7_Ready', 0.0, 0.0) == 0, if(SetVariable('Slot8_Name', 0.0, 0.0) == 0, if(SetVariable('Slot8_Team', 0.0, 0.0) == 0, if(SetVariable('Slot8_Ready', 0.0, 0.0) == 0, 0, 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0), 0)
```

**Effect 2 - Trigger broadcast (with delay for sync):**
```
SetTask('broadcast_lobby', 'Active', 0.5)
```

**Effect 3 - Reset:**
```
SetTask('clear_all_slots', 'NotActive', 0.1)
```

**reset_my_lobby Task (Single Player):**

Called by iframe when it detects its slot was cleared externally.

**Effect 1 - Reset local variables:**
```
SetVariable('My_Slot', 0.0, 0.0) + SetVariable('My_Team_Local', 0.0, 0.0)
```

**Effect 2 - Reset:**
```
SetTask('reset_my_lobby', 'NotActive', 0.1)
```

## Local Timer System (v3.0)

The leaderboard HUD now uses a **local timer** instead of relying on server-side `Elapsed_Seconds`:

### How It Works
1. `start_game` task sends `game_start` message to iframes
2. HUD starts local 10-minute countdown
3. On timer expiry, HUD compares scores and sends `timer_red_wins`, `timer_blue_wins`, or `timer_tie_wins`
4. On game end, `gameover_ready` sends `reset` message to stop timer

### Benefits
- No need for `Elapsed_Seconds`, `Timer_Host_Exists`, `I_Am_Host` variables
- No need for `GameTimer`, `DelayedHostCheck`, `BecomeHost` tasks
- No timer acceleration when multiple players join
- Simpler reset logic

### Backwards Compatibility
- Still accepts `time_` and `sync_` messages (for legacy setups)
- If `time_` received before `game_start`, will auto-start local timer

## Stale Data Handling

The HUD uses a `gameStartedClean` flag to filter stale scores from previous games:
- Rejects scores >= 20 until first legitimate low score seen
- Prevents "51" showing at start of new game

## Cache Busting

Always increment `?v=X` in iframe URLs when pushing updates. GitHub Pages caches aggressively.

## Future Ideas (Not Implemented)
- Kill feed
- Medal popups (Double Kill, etc.)
- Scoreboard (TAB to view K/D/A)
- Announcer text ("RED TAKES THE LEAD")
- Map intro screen
- MVP screen

These require kill tracking/credit system which is coming soon to Portals.
