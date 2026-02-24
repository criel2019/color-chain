# Color Chain Foundation (Multiplayer + Localization)

This fork is now prepared for future expansion in two directions:

- Localization (multi-language UI/runtime text)
- Competitive multiplayer (Tetris 99-style garbage exchange)

## What was added

- Runtime i18n dictionary and locale switcher (`en`, `ko`, `ja`, `es`) with English fallback
- Localized UI labels + runtime effect text (`CHAIN`, `BOOM`, `JUNK`, points, merge text)
- `window.ColorChain` integration API for embedding and future network layer
- Event hooks for gameplay milestones (`input`, `merge`, `explosionResolve`, `outgoingAttack`, etc.)
- Incoming attack queue (`queueIncomingAttack`) with delayed warning integration
- Seedable gameplay RNG (`seed`) for piece/junk randomness (effects remain cosmetic-random)

## Public API (current)

`window.ColorChain`

- `setLocale(locale)` / `getLocale()`
- `setMode(mode, options)` / `getMode()`
- `start(options)`
- `sendAction(action)`
- `queueIncomingAttack(packet)`
- `getSnapshot()`
- `on(event, handler)` / `off(event, handler)`

### `start(options)` examples

```js
ColorChain.start({ locale: 'en' });
ColorChain.start({ mode: 'versus', seed: 12345, playerId: 'p1' });
ColorChain.start({
  mode: 'versus',
  roomId: 'room-42',
  playerId: 'p1',
  outgoingAttack: (packet) => socket.emit('attack', packet)
});
```

## Multiplayer direction (recommended)

### Short-term (fastest)

- Client-authoritative input + server relay
- Server validates packet shape and match membership
- Server distributes `queueIncomingAttack({ cols: [...] })` packets with explicit columns

Why explicit `cols` now:
- Current code can consume `power`, but explicit `cols` avoids RNG divergence across clients.

### Mid-term (safer)

- Server-authoritative match state
- Clients send actions only (`left/right/rot/drop/hold`)
- Server simulates, emits state deltas / snapshots / attacks

## Tetris 99-style battle features to add next

- Targeting system (random / attacker / manual target / KOs)
- Garbage canceling (outgoing attack offsets incoming queue)
- Attack table tuning by chain/explosion size
- KO badges / bonus multipliers
- Spectator / replay serialization
- Match countdown + synchronized seed distribution

## Localization direction (recommended)

- Move `I18N` dictionary to separate `locales/*.json` files once build tooling is added
- Add string linting (missing key / fallback detection)
- Add RTL support check if Arabic/Hebrew planned
- Localize tutorial/rules and settings (next step)

## Important current limitation

- Gameplay and rendering are still in one HTML file. This is fine for prototyping, but networking will be easier after splitting into:
- `core/game-logic.js`
- `ui/render.js`
- `ui/i18n.js`
- `net/match-client.js`
