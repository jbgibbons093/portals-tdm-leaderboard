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

| Message | Format | Example |
|---------|--------|---------|
| Sync all data | `sync_TIME_RED_BLUE` | `sync_120_25_18` |
| Red score only | `red_SCORE` | `red_25` |
| Blue score only | `blue_SCORE` | `blue_18` |
| Time only | `time_ELAPSED` | `time_120` |
| Player slot | `my_slot_X` | `my_slot_3` |
| Lobby sync | `lobbysync_C1_T1_R1_C2_T2_R2...` | 8 slots × 3 values |
| Countdown | `countdown_X` | `countdown_3` |
| Scores (game-over) | `scores_RED_BLUE` | `scores_50_32` |
| Reset | `reset` | `reset` |

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
| `timer_red_wins` | Auto | Red reached 50 or time ran out with lead |
| `timer_blue_wins` | Auto | Blue reached 50 or time ran out with lead |
| `leaderboard_ready` | On HUD load | HUD initialized |

## Portals Setup Requirements

### Variables (Multiplayer)
- `Red_Score` - Red team points
- `Blue_Score` - Blue team points
- `Elapsed_Seconds` - Game timer
- `Game_In_Progress` - 1 during gameplay, 0 otherwise
- `Slot1_Claimed` through `Slot8_Claimed` - Lobby slot tracking
- `Slot1_Team` through `Slot8_Team` - Team assignments
- `Slot1_Ready` through `Slot8_Ready` - Ready status

### Variables (Single Player)
- `Player_Team` - This player's team (0=none, 1=red, 2=blue)
- `My_Slot` - This player's lobby slot (1-8)

### Key Tasks

**team_select_ready**
- Trigger: Iframe Message Received
- On Completed: Trigger `claim_slot` to assign player their slot
- Reset: Set back to NotActive when player enters lobby

**SyncToHUD** (looping, host-only)
- Sends `sync_|Elapsed_Seconds|_|Red_Score|_|Blue_Score|` every second

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
