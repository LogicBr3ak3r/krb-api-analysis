# Kingdom Rush Battles - API Exploitation Analysis Report

**Author:** LogicBr3ak3r
**Contact:** Telegram - [@LogicBr3ak3r](https://t.me/LogicBr3ak3r)

> **Note on Tools & Scripts:** All scripts and automation tools developed during this research - including `auto_farm.py` (the full economy farming bot), the PvP headless client, the savegame editor, and supporting utilities - will be released publicly once these reports gain sufficient attention. Follow the Telegram contact above to stay notified.

**Server**: `https://api.mp-ironhidegames.com`  
**Backend**: Express.js on nginx (AWS)  
**Auth**: JWT (HS256) Bearer tokens  
**CDN**: CloudFront → S3 (`d31ix1hmhuinz8.cloudfront.net`)  
**Rate Limit**: 500 requests per window (very permissive)

---

## 1. COMPLETE ENDPOINT MAP

| # | Method | Path | Purpose | Risk Level |
|---|--------|------|---------|------------|
| 1 | `HEAD` | `/healthy` | Health check / keepalive (every ~4s) | None |
| 2 | `GET` | `/remoteconfig/` | Feature flags, remote config | Low |
| 3 | `POST` | `/users/user-token` | Authentication / login | **CRITICAL** |
| 4 | `POST` | `/user-actions/user/` | Fetch user actions, missions, progress | Medium |
| 5 | `POST` | `/matches/new-or-update-game` | Create/start a match | **CRITICAL** |
| 6 | `POST` | `/matches/finish-game` | Report match result | **CRITICAL** |
| 7 | `POST` | `/logic/claim-many-mission-token` | Claim mission rewards | **HIGH** |
| 8 | `GET` | `/logic/level-up-card/{cardId}/` | Upgrade a card | **HIGH** |
| 9 | `GET` | `/logic/events-ui/{userId}` | Fetch events (Gacha, Onslaught, CoinFarm) | Medium |
| 10 | `GET` | `/logic/daily-missions-ui` | Fetch daily/weekly missions & roads | Medium |
| 11 | `GET` | `/admin-settings/` | **Full server config exposed** | **CRITICAL** |
| 12 | `GET` | `/chests-rewards/chests-reward-by-arena` | Chest cycle, loot tables, bravery chest | Medium |
| 13 | `GET` | `/in-app-purchase/filtered-for-user/` | All IAP offers for the user | Medium |
| 14 | `POST` | `/in-app-purchase/product-ids` | List of all Google Play store IDs | Low |
| 15 | `GET` | `/inbox/status` | Inbox message count | Low |
| 16 | `GET` | `/ServerBundles/Android/1.7.0.102/Retina/catalog_*.hash` | Asset bundle hash (CDN) | Low |

---

## 2. MATCH RESULT SPOOFING (CRITICAL)

### 2A. `POST /matches/new-or-update-game` - Start Match

**Client-controlled request body:**
```json
{
  "_id": null,
  "userId": "<USER_ID>",
  "userDeck": "cardId1, cardId2, ..., cardId8",
  "matchType": "TROPHYROAD",
  "arena": "664ced9c65ec847d7e4de6fa",
  "idPhoton": "<UUID>",
  "isBot": "true",
  "trophiesBot": 286,
  "medalsBot": 5,
  "win": null,
  "survivor": false,
  "unbreakable": false,
  "conqueror": false,
  "blitz": false,
  "slayer": false,
  "challenger": false,
  "stars": 0,
  "trophies": 0,
  "gold": 0,
  "realStarsEarned": 0,
  "medalsWins": 0,
  "tutorialStars": false,
  "eventRewards": null
}
```

**Exploitation angles:**
- `matchType` - client sends the match type. Could inject `"COLISEUM"`, `"ONSLAUGHT"`, `"COINFARM"`, `"GACHA"` etc.
- `isBot` - client declares if opponent is a bot. Force `"true"` to always play against bots
- `trophiesBot` - client sets the bot's trophy count (affects trophy gain/loss calculation)
- `medalsBot` - client sets bot medals
- `arena` - client chooses the arena ID, could be changed to higher arena for better rewards
- `stars`, `trophies`, `gold` - sent as 0 at start but fields exist for the server to potentially trust
- `eventRewards` - null at start, but the field exists
- `idPhoton` - Photon room ID, could be fabricated (no real Photon session needed for bot matches)

### 2B. `POST /matches/finish-game` - Report Match Result

**Client-controlled request body:**
```json
{
  "userId": "<USER_ID>",
  "userDeck": "cardId1, ..., cardId8",
  "arena": "664ced9c65ec847d7e4de6fa",
  "idPhoton": "<UUID>",
  "win": "false",
  "survivor": false,
  "unbreakable": false,
  "conqueror": false,
  "blitz": false,
  "slayer": false,
  "challenger": false,
  "tutorialStars": false,
  "reason": null,
  "missions": {
    "towers": { "buildCount": 2, "upgradeCount": 0, "skillCount": 0 },
    "spells": { "damageDealt": 0, "castCount": 0, "unitSpawnedCount": 0 },
    "hero": { "damageDealt": 919 },
    "creeps": { "deathCount": 15 }
  }
}
```

**Server response confirms these fields are TRUSTED:**
```json
{
  "stars": 0,
  "trophies": -9,
  "gold": 3,
  "starsResult": {
    "survivor": 0, "unbreakable": 0, "conqueror": 0,
    "blitz": 0, "slayer": 0, "challenger": 0,
    "tutorial": 0, "tutorialStars": 0
  },
  "medalsWins": 0
}
```

**Exploitation angles:**
- **`win`** - Change `"false"` → `"true"` to report a win. Server awards trophies (+25-30 per win vs -9 per loss)
- **Star achievement booleans** - `survivor`, `unbreakable`, `conqueror`, `blitz`, `slayer`, `challenger` - all client-reported. Setting ALL to `true` = **6 stars per match** (max possible). Stars are used to open chests
- **`missions` block** - All mission progress values are CLIENT-REPORTED:
  - `towers.buildCount` - inflate to instantly complete TowerBuild mission
  - `towers.upgradeCount` - inflate for TowerUpgrade mission
  - `spells.damageDealt` - inflate for SpellDamage mission (maxAmount: 30000)
  - `hero.damageDealt` - inflate for HeroDamage mission (maxAmount: 15000)
  - `creeps.deathCount` - inflate for DefeatCreeps mission (maxAmount: 200)
- **`tutorialStars`** - bool that may grant extra tutorial bonus stars
- **`arena`** - could be changed to a higher arena to get credit for higher-arena wins

### 2C. Full Match Spoofing Pipeline

1. `POST /matches/new-or-update-game` with `isBot: true`, `matchType: "TROPHYROAD"`
2. Wait ~5 seconds (match minimum duration)
3. `POST /matches/finish-game` with `win: "true"`, all star booleans `true`, inflated mission stats

**Expected yield per fake match:**
- +25-30 trophies
- +6 stars (enough to open a Silver chest instantly)
- +3-5 gold
- +20 playerXP
- Mission progress acceleration (complete all daily missions in 1-2 matches)

---

## 3. MISSION & REWARD MANIPULATION (HIGH)

### 3A. `POST /logic/claim-many-mission-token` - Claim Missions

**Request:**
```json
{ "missionIds": ["<MISSION_ID>"] }
```

**Response:** Full user object with updated currencies (gold, gems, stars, etc.)

**Exploitation:**
- Client chooses which `missionIds` to claim - could try claiming the same mission multiple times
- Could try claiming missions that aren't yet at `maxAmount` (status still "Pending")
- Could try claiming missions with fabricated IDs
- After match spoofing inflates all mission progress, claim all 12 daily missions rapidly

### 3B. Daily Mission Types (from traffic)

| Mission Type | maxAmount | Token Reward |
|---|---|---|
| LoginsDone | 1 | Tokens |
| PlayGames | 2 | Tokens |
| DefeatCreeps | 200 | Tokens |
| TowerBuild | 20 | Tokens |
| TowerUpgrade | 30 | Tokens |
| SpellDamage | 30000 | Tokens |
| HeroDamage | 15000 | Tokens |
| ChestOpen | 3 | Tokens |
| StarsEarn | 6 | Tokens |
| PurchaseDailyDeal | 2 | Tokens |
| SpendGems | 18 | Tokens |
| UnlockChestWithGems | 1 | Tokens |

### 3C. Mission Road Rewards

The `daily-missions-ui` response reveals a mission road system:
- Accumulate mission tokens to reach milestones (e.g., `amountRequirement: 20`)
- Milestones grant rewards like stars, gems, scrolls, chests
- `missionRoadRewardsClaimedDaily` and `missionRoadRewardsClaimedWeekly` track claims
- `trophyRoadRewardsClaimed` tracks trophy road milestone claims

---

## 4. CARD MANIPULATION (HIGH)

### 4A. `GET /logic/level-up-card/{cardId}/`

**Observed:** `GET /logic/level-up-card/<CARD_ID>/`

**Response includes updated card:**
```json
{
  "card": {
    "_id": "<CARD_ID>",
    "name": "MG_Reinforcements",
    "code": " s2",
    "amount": 4,
    "level": 2,
    "referenceCardId": "659eda6d11552d01597f6d03",
    "type": "Spell",
    "wornCopies": 2
  }
}
```

**User gold went from 85 → 80 (cost: 5 gold per level-up)**

**Exploitation:**
- This is a `GET` request that performs a write operation (side effect) - unusual and potentially exploitable
- The card ID is in the URL path - try level-up on cards you don't own (using other users' card IDs)
- No request body validation needed - just call the endpoint
- Rapidly spam to level up cards if gold allows
- Gold cost is deducted server-side, but combined with gold farming this could rapidly max out cards

### 4B. Card System Details

From admin-settings:
- `maxLevelOfCards`: 15
- Card rarity to gold conversion rates are exposed:
  - Common → 5 gold
  - Rare → 30 gold  
  - Epic → 300 gold
  - Legendary → 10000 gold

---

## 5. ADMIN-SETTINGS INFORMATION LEAK (CRITICAL)

### `GET /admin-settings/`

**This endpoint exposes the ENTIRE server configuration including:**

```json
{
  "enviroment": "GD",
  "version": "1.7.0_p7_p7",
  "initialArena": "66b4bfa5260bca45bba326c2",
  "initialStage": "665f39f06124971ec5f5538b",
  "initialCards": ["659ed62511552d01597f6cfd", ...],
  "maxLevelOfCards": "15",
  "capTrophies": 3000,
  "capXp": 5653798,
  "gamesToWinRate": 1,
  "gamesRate": 10,
  "winRateLowerLimit": -3,
  "winRateUpperLimit": 3,
  "LegendaryToGold": 10000,
  "commonToGold": 5,
  "epicToGold": 300,
  "rareToGold": 30,
  "pvpCardLevelCap": 7,
  "pvpCreepLevel": 7,
  "pvpPlayerBoosterLevel": 6,
  "maxChasersAmount": 3,
  "changeNameCost": 5,
  "freeChangesName": 2,
  "starterPackLimitTrophies": 2300,
  "tutorialStars": 3
}
```

**Exploitation:**
- **Full matchmaking algorithm exposed**: `winRateLowerLimit`, `winRateUpperLimit`, `gamesRate`, `gamesToWinRate` - understand exactly how matchmaking works to game the system
- **Trophy cap**: 3000 - know the ceiling
- **All arena IDs**: Can enumerate and target specific arenas
- **All card reference IDs**: Know every card in the game
- **All shop offer IDs**: Know what IAPs exist
- **PvP level caps**: Know exact PvP constraints
- **Leaderboard timing**: `startLeaderboards`, `endLeaderboards`, `resetLeaderboards: "1M"` - know when to push
- **Daily deal reset timing**: `resetDailyDealsTimestamp`, `endDailyDealTimestamp`
- **Daily cron job status**: `dailyCronJobInProgress` - timing window for potential race conditions

---

## 6. CHEST & LOOT SYSTEM (MEDIUM-HIGH)

### 6A. `GET /chests-rewards/chests-reward-by-arena`

**Response reveals the ENTIRE chest cycle** (20,253 bytes):

Chest cycle pattern for Arena 1:
1. **Silver** (3 stars, instant open) - 1 common tower card, 15 gold
2. **Gold** (5 stars, instant open) - 4 common + 1 rare, 35 gold
3. **Silver Lock** (3 stars, 3h timer or 18 gems to skip)
4. **Silver** (instant)
5. **Magical** (7 stars, instant) - 8 common + 2 rare, 60 gold
6. **Silver** (instant)
7. **Gold** (instant)
8. **Silver** (instant)
9. **Silver** (instant)
10. **Gold Lock** (5 stars, 3h timer or 18 gems)
11. **Silver** (instant)
12. **Gold** (instant)
13. **Silver Lock** (3h timer or 18 gems)
14. **Silver Lock** (3h timer or 18 gems)

**Bravery Chest**: Opened every 12h after 5 matches (10 chests), contains 5 common + 2 rare + 35 gold

**Exploitation:**
- **Chest index manipulation**: User has `currentChestsIndex: 2`. If this value can be modified via any endpoint, could always target the Magical chest (index 4)
- **`nextChestStarsRequired: 0`**: Stars needed for next chest - if manipulable, always 0 = free chests
- **`nextChestWaiting`**: Timer-based locked chests - manipulating this timestamp could skip timers
- **`openInGems: 18`** for locked chests - but with gem manipulation, this is irrelevant
- **`matchesToBraveryChest: 2`**: Only 2 more matches needed - match spoofing accelerates this
- **`braveryOpenedToday: false`**: If manipulable, could reset to claim bravery chest multiple times
- **Star-based opening**: Combined with star spoofing from match results, can open chests infinitely

### 6B. Chest Loot Table Knowledge

The full loot table probabilities are exposed:
- `commonPercentageMin`/`Max`, `rarePercentageMin`/`Max` etc.
- `randomQuantityTower`, `randomQuantitySpell`, `randomQuantityHero`
- `quantityTopRareCards`, `quantityTopEpicCards`, `quantityTopLegendaryCards`

This allows calculating expected value per chest type to optimize farming strategy.

---

## 7. EVENT & GACHA EXPLOITATION (HIGH)

### 7A. Active Events (from `/logic/events-ui/`)

| Event | Type | Key Mechanic |
|---|---|---|
| Summon Rift Banner 11 Alleria Wildcat | GACHA | Summon Scrolls → cards/gems/gold |
| Coin Farm Arena 1 | COINFARM | Event tickets → play for coins (80 win/40 lose) |
| Onslaught | ONSLAUGHT | Event tickets → play for Summon Scrolls |
| Onslaught 2 | ONSLAUGHT | 4 tickets required, harder difficulty |

### 7B. Gacha System

The Gacha has a **pity system at threshold: 40** pulls, guaranteeing a specific card reward (Alleria Wildcat, level 5, 3 copies).

Gacha reward pool (per pull):
- SoftCoin: 100 or 200 gold
- HardCoin: 3 or 5 gems
- ChestsRewards: 2-20 random rarity cards (Common or Rare)
- CardReward: Featured card (pity)

**Exploitation:**
- Event progress (`progress`, `extraCounter`) tracked server-side but event participation may be spoofable via match type manipulation
- `matchType: "ONSLAUGHT"` in match start could register event participation
- `eventRewards` field in match results could be manipulated

### 7C. Coin Farm

- `softCoinWin: 80` / `softCoinLose: 40` - guaranteed gold either way
- `ticketsRequired: 1` per match
- `gemsToUnlock: 35` - costs gems to enter
- Combined with match spoofing: always report win for 80 gold per match

### 7D. Event Currencies

User state shows:
- `summonScrolls: 0` - for Gacha pulls
- `eventTickets: 0` - for event entry
- `medals: 0` - for PvP/leaderboard

---

## 8. IN-APP PURCHASE / SHOP SYSTEM (HIGH)

### 8A. `GET /in-app-purchase/filtered-for-user/`

Returns ALL available purchases (36,314 bytes). Types found:

**Gem Shop (real money → gems):**
| Offer | Price | Gems |
|---|---|---|
| Handful of Gems | $1.99 | 100 |
| Bunch of Gems | $4.99 | 300 |
| Barrel of Gems | $9.99 | 750 |
| Chest of Gems | $19.99 | 2000 |
| Wagon of Gems | $49.99 | 5500 |
| Mountain of Gems | $99.99 | 12000 |

**Gold Shop (gems → gold):**
| Offer | Cost | Gold |
|---|---|---|
| Handful of Coins | 60 gems | 1,000 |
| Chest of Coins | 500 gems | 10,000 |
| Mountain of Coins | 4,500 gems | 100,000 |

**Chests (gems → chests):**
| Chest | Cost | Contents |
|---|---|---|
| Soldier Chest | 250 gems | Arena-specific chest rewards |
| Captain Chest | 800 gems | Better arena-specific rewards |
| King Chest | 3,000 gems | Best arena-specific rewards |

**Daily Deals:**
- `HardCoin 5` - Free daily 5 gems (cost: 0)
- `ChestsRewards` cards - 20-80 gold (soft coin)
- `purchased: false` flag tracked per deal

**Special Offers (IAP):**
- Mega Gems Offer: $9.99 → 1,200 gems (60x value multiplier!)
- Mega Coins Offer: $6.99 → 9,000 coins (25x value multiplier!)
- Starter Packs: $6.99-$19.99 (cards + gems + gold)
- Season Offers: $34.99-$49.99 (LegendaryAwakening, EpicAwakening)
- Summon Scroll packs: 150 gems → 10 scrolls, or $14.99-$19.99 → 60-100 scrolls

**Exploitation:**
- **Free daily gems**: The `HardCoin 5` daily deal costs 0 - claim daily without playing
- **Daily deal claims may not be rate-limited**: `costHardCoin: 0`, `costSoftCoin: 0` - try claiming repeatedly
- **`limitPurchases`**: Some offers have `limitPurchases: 0` (unlimited?), others have specific limits (1-3)
- **Purchase validation**: For real-money purchases, the server likely validates with Google Play receipt. But for **gem/gold purchases**, the server may trust the client's currency balance → combined with currency manipulation, buy everything for free
- **`purchased: false`**: This flag is in the response - potential to reset it or bypass checks

### 8B. `POST /in-app-purchase/product-ids`

Returns all 37 Google Play store product IDs. This maps every purchasable SKU.

---

## 9. AUTHENTICATION & SESSION WEAKNESSES (CRITICAL)

### 9A. JWT Token Structure

The JWT is HS256 signed:
```
Header: {"typ":"JWT","alg":"HS256"}
Payload: {
  "userId": "<USER_ID>",
  "playerId": "<PLAYER_ID>",
  "type": "player",
  "iat": 1772299730,
  "exp": 1772321330
}
```

**Observations:**
- Token expires in ~6 hours (21600 seconds)
- **Token is refreshed on EVERY API call** - the `user.token` field in responses contains a new JWT with updated `iat`/`exp`. This means the session never truly expires if you keep making calls
- **HS256 algorithm** - vulnerable to brute-force if secret is weak. If secret is discovered, can forge tokens for any user
- **No `aud` (audience) or `iss` (issuer)** in the game JWT (though the Unity token has these)

### 9B. `POST /users/user-token` - Login

**Request:** URL-encoded form data:
```
playerId=<PLAYER_ID>
&unity-token=<Unity Services JWT (RS256)>
&provider=google
```

**Weaknesses:**
- `provider` field is client-set - could try `"guest"`, `"google"`, `"apple"`, `"facebook"`, etc.
- The `playerId` is sent by the client - potential for ID enumeration/spoofing if Unity token validation is weak
- Unity token is RS256 (more secure), but the game's own token is HS256

### 9C. Token Reuse

- The same token is used across all endpoints
- No per-request nonce or CSRF protection
- `Access-Control-Allow-Origin: *` - any website can make requests to this API
- Captured tokens can be replayed from any tool (curl, Python requests, etc.) for ~6 hours

---

## 10. RATE LIMITING GAPS (HIGH)

### 10A. Observed Rate Limits

Every response includes:
```
X-RateLimit-Limit: 500
X-RateLimit-Remaining: 499
X-RateLimit-Reset: <unix_timestamp>
```

**Analysis:**
- **500 requests per window** - extremely permissive
- Reset window appears to be ~60 seconds
- Rate limit is per-IP, not per-endpoint
- No exponential backoff or ban mechanism observed
- The health check (`/healthy`) consumes rate limit tokens but fires every ~4 seconds

**Exploitation:**
- At 500 req/min, can execute ~8 complete match cycles per minute (new-game + finish-game)
- Over 1 hour: ~480 spoofed matches
- At +30 trophies per win: **14,400 trophies/hour** (cap is 3000, reached in ~12 minutes)
- At +6 stars per win: **2,880 stars/hour** (enough to open every chest type)
- At +3-5 gold per win: **2,400 gold/hour** (plus chest rewards)

### 10B. No Duplicate Request Prevention

- No idempotency keys on POST requests
- No nonce or request-ID tracking
- The same `claim-many-mission-token` request could potentially be replayed
- Same `finish-game` could potentially be called multiple times for the same match

---

## 11. SERVER TRUST ISSUES (CRITICAL)

### 11A. Fields the Client Reports That Server Trusts

| Field | Endpoint | What It Controls |
|---|---|---|
| `win` | finish-game | Win/loss outcome |
| `survivor` | finish-game | Star bonus |
| `unbreakable` | finish-game | Star bonus |
| `conqueror` | finish-game | Star bonus |
| `blitz` | finish-game | Star bonus |
| `slayer` | finish-game | Star bonus |
| `challenger` | finish-game | Star bonus |
| `missions.towers.*` | finish-game | Mission progress |
| `missions.spells.*` | finish-game | Mission progress |
| `missions.hero.*` | finish-game | Mission progress |
| `missions.creeps.*` | finish-game | Mission progress |
| `isBot` | new-or-update-game | Bot match flag |
| `trophiesBot` | new-or-update-game | Bot trophy level |
| `medalsBot` | new-or-update-game | Bot medal count |
| `matchType` | new-or-update-game | Match category |
| `arena` | new-or-update-game | Arena selection |
| `missionIds` | claim-many-mission-token | Which missions to claim |

### 11B. Evidence of Server-Side Computation

Some values ARE computed server-side:
- Trophy change amount: -9 for loss (calculated from trophies delta)
- Gold reward: 3 gold for a loss (small consolation)
- Match duration: `userMatchDuration: 68` (calculated from start/end timestamps)
- Stars result: Mapped from the client-reported booleans

The server DOES calculate rewards based on client-reported values rather than validating gameplay. For bot matches, there's no Photon server to verify the actual game outcome.

---

## 12. TIMING EXPLOITS (MEDIUM)

### 12A. Chest Timers

- **`nextChestWaiting`** - ISO timestamp; if client can send a modified timestamp or if this is checked client-side, timers can be bypassed
- **Locked chests (3h timer)**: `openInTime: "3h"` - but `openInGems: 18` to skip. With gem farming, irrelevant
- **Bravery chest (12h timer)**: `openInTime: "12h"` - resets daily

### 12B. Daily Mission Reset

- `lastMissionsDailyTimestamp` - missions reset at 13:00 UTC daily
- `lastMissionsWeeklyTimestamp` - weekly reset on Wednesdays
- If the client can manipulate these timestamps (e.g., via `user-actions` POST), missions could be re-rolled

### 12C. Daily Deal Timing

- `resetDailyDealsTimestamp` - exposed in admin-settings
- `endDailyDealTimestamp` - deal expiry
- `timeToCreateDailyDeals: 1440` - 24 hours between deals

### 12D. Leaderboard Timing

- `startLeaderboards` / `endLeaderboards` - exposed in admin-settings
- `resetLeaderboards: "1M"` - monthly reset
- Knowing exact timing allows pushing trophies right before leaderboard snapshot

### 12E. Match Duration Minimum

The observed match lasted 68 seconds (19:31:06 → 19:32:15). The minimum viable match duration before the server accepts a finish-game needs testing. Could be as low as a few seconds for bot matches.

---

## 13. LEADERBOARD & RANKING ABUSE (HIGH)

### 13A. Trophy System

- Trophy gain example: after 5 games (4 wins / 1 loss), net gain ~+80-90 trophies
- Trophy gain per win: ~+25-30
- Trophy loss per loss: ~-9
- Trophy cap: 3000 (from admin-settings)
- Leaderboard IDs exposed in admin-settings

**Exploitation:**
- Spoof wins to reach 3000 trophies cap
- At ~30 trophies per spoofed win, need ~100 wins from 0
- At 8 matches/minute, reach cap in ~12.5 minutes
- Top leaderboard position → leaderboard rewards (referenced by reward ID)

### 13B. Win Rate Tracking

User state tracks:
```json
{
  "winRate": 0,
  "lastWinsRate": 0,
  "positiveStreak": 0,
  "negativeStreak": 1,
  "longestWinStreak": 4,
  "strikeCountPositive": 1,
  "strikeCountNegative": 0,
  "matchesResults": 0
}
```

Admin-settings reveals matchmaking protection:
```json
{
  "winRateLowerLimit": -3,
  "winRateUpperLimit": 3,
  "gamesToWinRate": 1,
  "gamesRate": 10
}
```

This suggests the server uses a win-rate system to adjust matchmaking difficulty, but since bot matches are client-controlled, this can be gamed entirely.

---

## 14. UNDISCOVERED ENDPOINTS (SPECULATIVE)

Based on patterns observed, these endpoints likely exist but weren't captured:

| Speculated Endpoint | Purpose |
|---|---|
| `POST /logic/open-chest` | Open a chest (not captured but chests were opened) |
| `POST /logic/claim-trophy-road-reward` | Claim trophy road milestones |
| `POST /logic/claim-daily-reward` | Daily login reward |
| `POST /logic/claim-mission-road-reward` | Mission milestone rewards |
| `POST /logic/claim-player-xp-reward` | XP level-up rewards |
| `POST /logic/gacha-pull` or `/logic/summon` | Gacha summon with scrolls |
| `POST /in-app-purchase/buy` or `/purchase` | Execute a shop purchase |
| `POST /in-app-purchase/daily-deal/buy` | Buy daily deal |
| `POST /user-actions/change-name` | Change username |
| `POST /logic/event/coin-farm/play` | Enter coin farm event |
| `POST /logic/event/onslaught/play` | Enter onslaught event |
| `GET /leaderboard/` | View leaderboard |
| `POST /inbox/read` or `/inbox/claim` | Read/claim inbox messages |
| `POST /user-actions/equip-emote` | Equip emotes |

---

## 15. ATTACK CHAIN SUMMARY

### Chain 1: Infinite Trophy/Star Farm
```
1. POST /users/user-token (authenticate)
2. Loop:
   a. POST /matches/new-or-update-game (isBot: true, matchType: TROPHYROAD)
   b. Sleep(5 seconds)
   c. POST /matches/finish-game (win: true, all stars: true, inflated missions)
   d. POST /logic/claim-many-mission-token (claim completed missions)
3. Reach 3000 trophies in ~12 minutes
4. All daily missions completed in 2-3 matches
5. All chests openable with accumulated stars
```

### Chain 2: Currency Farm
```
1. Authenticate
2. Spoof wins for gold + stars
3. Open chests for cards + gold + gems
4. Level up all cards with gold
5. Buy gold with gems from chests
6. Daily deal: claim free 5 gems daily
7. Repeat daily
```

### Chain 3: Event Exploitation
```
1. Authenticate
2. POST /matches/new-or-update-game with matchType: ONSLAUGHT/COINFARM
3. Finish with win: true → earn event progress + event currency
4. Claim event rewards
5. Gacha pull with earned scrolls
6. Reach pity threshold (40) for guaranteed legendary card
```

### Chain 4: Leaderboard Manipulation
```
1. Create multiple guest accounts (no email/verification needed)
2. Farm trophies on main account to 3000
3. Wait for leaderboard snapshot near endLeaderboards date
4. Claim leaderboard rewards
```

---

## 16. INFRASTRUCTURE NOTES

| Detail | Value |
|---|---|
| Server | nginx + Express.js (Node.js) |
| Database | MongoDB (ObjectId format in all `_id` fields) |
| CDN | AWS CloudFront → S3 |
| Multiplayer | Photon Quantum |
| Auth | Unity Authentication → Custom HS256 JWT |
| CORS | `Access-Control-Allow-Origin: *` (wide open) |
| Rate Limit | 500/window (~60s) |
| SSL | TLS |
| Region | Server appears multi-region, player region detected from IP |
| User-Agent | `UnityPlayer/2021.3.56f2 (UnityWebRequest/1.0, libcurl/8.10.1-DEV)` |

### Custom Headers Required

Every game request must include:
```
authorization: Bearer <JWT>
x-krb-protocol: 7
x-krb-version: 1.7.0
x-krb-platform: Android
x-krb-locale: en_US
X-Unity-Version: 2021.3.56f2
```

---

## 17. RISK ASSESSMENT

| Category | Severity | Confirmed? | Notes |
|---|---|---|---|
| Match result spoofing (win/loss) | **CRITICAL** | Yes (fields in request) | `win` boolean is client-sent |
| Star inflation | **CRITICAL** | Yes | 6 star booleans are client-sent |
| Mission progress inflation | **HIGH** | Yes | All mission stats client-reported |
| Admin settings leak | **CRITICAL** | Yes | Full config exposed to any auth'd user |
| JWT HS256 weakness | **MEDIUM** | Yes | Brute-forceable if weak secret |
| Rate limit bypass | **HIGH** | Yes | 500 req/min, no per-endpoint limit |
| Chest timer bypass | **MEDIUM** | Needs testing | Timer fields visible but manipulation method unclear |
| IAP receipt validation bypass | **MEDIUM** | Needs testing | In-game gem/gold purchases may not validate |
| Duplicate request replay | **HIGH** | Needs testing | No idempotency protection observed |
| CORS misconfiguration | **MEDIUM** | Yes | `Access-Control-Allow-Origin: *` |
| Event participation spoofing | **HIGH** | Needs testing | matchType is client-controlled |
| Leaderboard manipulation | **HIGH** | Yes (via trophy farming) | Trophy cap at 3000 reachable in minutes |
| User enumeration | **LOW** | Needs testing | MongoDB ObjectIds are sequential |
