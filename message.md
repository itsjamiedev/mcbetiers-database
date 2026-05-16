# DATABASE.md — MCBE Tiers Platform
### How the Data Layer Works, From First Join to Global Ranking

---

## Preface

Every number you see on the leaderboard, every rank badge earned, every verified player in the system — it all flows through a single PostgreSQL database hosted on Supabase. This document tells the full story of that journey: from the moment a player first joins the Minecraft server, all the way to their name appearing at the top of the global standings.

---

## Chapter 1 — The First Contact: A Player Joins the Server

Picture this. A player boots up Minecraft Bedrock, connects to the MCBE Tiers server for the first time, and the world loads around them. In that exact moment, the Verify Plugin running on the server springs to life.

The plugin reads two pieces of identity from the game client automatically — the player's **IGN** (In-Game Name, their username) and their **XUID** (Xbox User ID, Microsoft's globally unique identifier for every Bedrock account). These two values are the player's permanent identity. The IGN can theoretically change; the XUID never will.

The plugin then generates a **short random verification code** — something like `VR-4821` — and displays it to the player in chat:

```
Welcome to MCBE Tiers!
To verify your account, run /verify in our Discord with code: VR-4821
This code expires in 10 minutes.
```

The moment that code is generated, the plugin fires an HTTP request to the Supabase REST API and writes a new row into the `verification_codes` table.

---

## Chapter 2 — The Staging Table: `verification_codes`

```sql
CREATE TABLE public.verification_codes (
    id         UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    code       TEXT         NOT NULL,
    discord_id TEXT         NOT NULL,
    ign        TEXT,
    xuid       TEXT,
    fuid       TEXT,
    region     TEXT,
    device     TEXT,
    type       TEXT         DEFAULT 'register',
    guild_id   TEXT,
    status     TEXT         DEFAULT 'pending',
    created_at TIMESTAMPTZ  DEFAULT NOW()
);
```

Think of this table as a **temporary holding room** — a staging area where identity fragments wait to be assembled into a complete player profile. When the plugin writes the code, it knows the player's `ign`, `xuid`, and `device`. It also detects the player's network region based on their connection latency, and stores that as `region` (either `NA`, `EU`, or `AS`).

The `fuid` (Functional Unique ID) is a composite identifier generated from the player's device fingerprint and session data — an extra layer of identity that persists even across account resets.

The `status` is set to `pending`. The code is live. The clock is ticking.

---

## Chapter 3 — The Discord Bridge: The Verify Bot Completes the Link

The player switches to Discord and runs:

```
/verify VR-4821
```

The **Verify Bot** receives this slash command. It looks up the command invoker's Discord User ID — this is now the `discord_id`. Armed with that, it reaches into the `verification_codes` table and queries:

```sql
SELECT * FROM verification_codes
WHERE code = 'VR-4821'
  AND status = 'pending'
ORDER BY created_at DESC
LIMIT 1;
```

If the code is found and not expired, the bot updates the row with the `discord_id` from the Discord interaction, and flips `status` to `'verified'`.

Now all five identity pillars are assembled in a single row:
- `ign` — their Minecraft username
- `xuid` — their immutable Microsoft account ID
- `discord_id` — their Discord User ID
- `device` — what platform they play on (MOBILE, PC, CONSOLE, KEYBOARD)
- `region` — where they play from (NA, EU, AS)

---

## Chapter 4 — Birth of a Player: The `users` Table

The moment `status` flips to `'verified'`, the bot triggers the final act: provisioning the player's permanent record.

```sql
CREATE TABLE public.users (
    id              UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    username        TEXT        UNIQUE NOT NULL,
    xuid            TEXT        UNIQUE,
    discord_id      TEXT,
    overall_elo     INTEGER     DEFAULT 0,
    overall_ranking INTEGER,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    last_active     TIMESTAMPTZ DEFAULT NOW(),
    region          TEXT,
    fuid            TEXT,
    device          TEXT
);
```

The bot `INSERT`s a new row into `users`, copying over everything from the verified code row. The player now has a **UUID** — their permanent internal identity within the MCBE Tiers ecosystem. From this moment forward, every rank, every ELO score, every leaderboard position is tied to this UUID.

The `verification_codes` row served its purpose. It can be cleaned up or archived. The player has graduated from staging into the live system.

---

## Chapter 5 — The Three Kingdoms: Ranking Tables

MCBE Tiers doesn't have one leaderboard — it has three, each representing a different category of Minecraft Bedrock PvP. The moment a player is created in `users`, three corresponding rows are inserted — one in each ranking table — all referencing the player's UUID via foreign key.

### 5.1 — `geyser_rankings` (Crossplay / Geyser Players)

```sql
CREATE TABLE public.geyser_rankings (
    user_id    UUID  PRIMARY KEY REFERENCES public.users(id) ON DELETE CASCADE,
    sword_elo  INTEGER DEFAULT 0,
    axe_elo    INTEGER DEFAULT 0,
    mace_elo   INTEGER DEFAULT 0,
    crystal_elo INTEGER DEFAULT 0,
    nethpot_elo INTEGER DEFAULT 0,
    smp_elo    INTEGER DEFAULT 0,
    uhc_elo    INTEGER DEFAULT 0,
    spear_elo  INTEGER DEFAULT 0,
    total_elo  INTEGER DEFAULT 0,
    ranking    INTEGER
);
```

These are players who connect via the **Geyser protocol** — Bedrock clients competing in Java-style gamemodes. Every mode — Sword, Axe, Mace, Crystal, NethPot, SMP, UHC, Spear — has its own ELO score. `total_elo` is the aggregate, and `ranking` is their position on the Geyser-specific leaderboard.

### 5.2 — `native_rankings` (Pure Bedrock Players)

```sql
CREATE TABLE public.native_rankings (
    user_id      UUID  PRIMARY KEY REFERENCES public.users(id) ON DELETE CASCADE,
    bedwars_elo  INTEGER DEFAULT 0,
    boxing_elo   INTEGER DEFAULT 0,
    bridge_elo   INTEGER DEFAULT 0,
    midfight_elo INTEGER DEFAULT 0,
    nodebuff_elo INTEGER DEFAULT 0,
    skywars_elo  INTEGER DEFAULT 0,
    sumo_elo     INTEGER DEFAULT 0,
    total_elo    INTEGER DEFAULT 0,
    ranking      INTEGER
);
```

These are **native Bedrock** players — no crossplay bridge. Their gamemodes reflect the classic MCBE competitive scene: Bedwars, Sumo, Bridge, Nodebuff, Skywars, Boxing, and Midfight.

### 5.3 — `sub_rankings` (Diamond-tier Game Modes)

```sql
CREATE TABLE public.sub_rankings (
    user_id         UUID  PRIMARY KEY REFERENCES public.users(id) ON DELETE CASCADE,
    dia_smp_elo     INTEGER DEFAULT 0,
    dia_pot_elo     INTEGER DEFAULT 0,
    dia_crystal_elo INTEGER DEFAULT 0,
    total_elo       INTEGER DEFAULT 0,
    ranking         INTEGER
);
```

A specialized competitive bracket for diamond-tier variants: Dia SMP, Dia Pot, and Dia Crystal — the highest-stakes subset of competitive play.

### 5.4 — The Match Logs: `geyser_matches` & `native_matches`

While the ranking tables store the current *state* of a player, these two tables store the *history* of every duel. 

**IMPORTANT:** Because these tables use foreign key constraints (`REFERENCES public.users(id)`), the system enforces a strict rule: **Only verified players can have match data recorded.** If a player hasn't completed the verification journey described in Chapter 3 and 4, they don't exist in the `users` table, and the database will reject any attempt to log a match involving them.

```sql
-- Example Schema (Geyser)
CREATE TABLE public.geyser_matches (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    winner_id UUID REFERENCES public.users(id) ON DELETE CASCADE,
    loser_id UUID REFERENCES public.users(id) ON DELETE CASCADE,
    gamemode TEXT NOT NULL,
    winner_elo_change INTEGER NOT NULL,
    loser_elo_change INTEGER NOT NULL,
    winner_old_elo INTEGER,
    loser_old_elo INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

These tables are designed for deep analysis. We don't just store who won; we store the **full context of the duel**:
- **The Combatants:** Both the `winner_id` and the `loser_id` (the opponent) are tracked, allowing for head-to-head winrate calculations.
- **The Stakes:** We record the `old_elo` for both players *before* the fight, and the specific `elo_change` that occurred as a result.
- **The Arena:** The `gamemode` tells us exactly what style of combat took place.

This is the foundation for the "Match History" section of player profiles, showing a timeline of who they fought, what they won, and how their ELO evolved fight-by-fight.

---

## Chapter 6 — The ELO Engine: How Rankings Move

Every time a player wins or loses a match, the system writes to `elo_updates` — a full audit log of every ELO transaction ever made.

```sql
CREATE TABLE public.elo_updates (
    id         BIGSERIAL PRIMARY KEY,
    user_id    UUID      NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    category   TEXT      NOT NULL,  -- 'geyser' | 'native' | 'sub'
    gamemode   TEXT      NOT NULL,  -- 'sword', 'bedwars', etc.
    old_elo    INTEGER   NOT NULL,
    new_elo    INTEGER   NOT NULL,
    elo_change INTEGER   NOT NULL,  -- positive = win, negative = loss
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

Nothing is ever overwritten silently. Every ELO movement — whether a +18 win or a -12 loss — is permanently recorded here. This table is the **source of truth for competitive integrity**: suspicious ELO patterns, rollback requests, dispute resolution — all investigated through this log.

After the `elo_updates` row is written:

1. The corresponding ranking table (`geyser_rankings`, `native_rankings`, or `sub_rankings`) is updated with the new ELO value and recalculated `total_elo`
2. The `users.overall_elo` is recalculated as the aggregate across all applicable categories
3. `users.overall_ranking` is recomputed
4. `users.last_active` is updated to `NOW()`

The leaderboard reflects reality within seconds.

---

## Chapter 7 — The Global Rankings: How the Leaderboard is Built

The leaderboard you see at `mcbetiers.com/leaderboard` pulls from a single view: players sorted by `users.overall_elo` descending, with `overall_ranking` as the position number.

The website's Cloudflare Pages Function (`/v1/leaderboard/overall`) queries Supabase via the REST API using the `service_role` key, which bypasses Row Level Security, and returns the paginated result set. The frontend renders each player card with:

- Their `username` from `users`
- Their `overall_elo` and `overall_ranking` from `users`
- Their `region` and `device` from `users`
- Their skin render fetched live from `https://api.mcbetiers.com/body/{username}`

---

## Chapter 8 — The Staff Layer: Controlled Access

Running parallel to all of this is a separate authentication system, entirely isolated from the player tables.

```sql
CREATE TABLE public.staff (
    id            UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    ign           TEXT        UNIQUE NOT NULL,
    role          TEXT        NOT NULL DEFAULT 'staff',  -- staff | admin | owner
    password_hash TEXT        NOT NULL,
    session_token TEXT,
    created_at    TIMESTAMPTZ DEFAULT NOW(),
    last_login    TIMESTAMPTZ
);
```

Staff accounts are provisioned manually. Passwords are hashed using **bcrypt** (via PostgreSQL's `pgcrypto` extension — specifically `crypt()` + `gen_salt('bf', 12)`). No plaintext is ever stored anywhere.

When a staff member logs in via the Staff Panel at `mcbetiers.com/staff`, the login API calls a `SECURITY DEFINER` RPC — `check_staff_password` — which runs with elevated database privileges, bypassing RLS, and returns the staff record only if the bcrypt hash matches. A session token is then generated, stored in `staff.session_token`, and set as an `HttpOnly` cookie.

---

## Chapter 9 — Security Architecture

Every table in the schema has **Row Level Security (RLS) enabled**. The default posture is:

| Role | Permission |
|---|---|
| `service_role` | Full read + write on all tables |
| `anon` / `authenticated` | Read-only (SELECT) on public tables |
| `anon` / `authenticated` | No direct write access to any table |

All mutations (ELO updates, player creation, staff session writes) go through **Cloudflare Pages Functions** using the `service_role` key, which is never exposed to the browser. The frontend only ever receives a filtered, read-only view of the data it needs.

Sensitive operations like password verification use `SECURITY DEFINER` RPCs, which execute in the database engine's elevated context — meaning even if the caller only has `anon` permissions, the function itself can read the `staff` table's `password_hash` column without that data ever leaving the database.

---

## Data Flow Summary

```
Player joins MC server
        │
        ▼
Verify Plugin generates code
        │
        ▼
verification_codes (status: pending)
        │
        ▼ Player runs /verify in Discord
Verify Bot matches code + discord_id
        │
        ▼
verification_codes (status: verified)
        │
        ▼
users row created (UUID assigned)
        │
        ├──▶ geyser_rankings row (all ELOs = 0)
        ├──▶ native_rankings row (all ELOs = 0)
        └──▶ sub_rankings row (all ELOs = 0)
                    │
                    ▼ After matches played
              elo_updates log entry
                    │
                    ▼
           ranking table updated
                    │
                    ▼
       users.overall_elo recalculated
                    │
                    ▼
         Leaderboard reflects result
```

---

*Last updated: May 2026 — MCBE Tiers Platform*
