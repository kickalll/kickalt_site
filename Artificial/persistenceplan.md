# Persistence Plan

Documents what needs to be added to `index.html` when persistence is desired. Currently all state is in-memory and resets on reload.

---

## State to persist

| Variable | localStorage key | Type | Default |
|---|---|---|---|
| `fishPoints` | `fishi_points` | number | `0` |
| `fishCanTalkUnlocked` | `unlock_fish_can_talk` | `'1'` / absent | `false` |

---

## Changes required

### Load on IIFE init (replace the current `let fishPoints = 0` and `let fishCanTalkUnlocked = false`)

```js
let fishPoints = parseInt(localStorage.getItem('fishi_points') || '0', 10) || 0;
let fishCanTalkUnlocked = localStorage.getItem('unlock_fish_can_talk') === '1';
```

### Save points on change (inside `addFishPoints`, after `fishPoints += amount`)

```js
localStorage.setItem('fishi_points', fishPoints);
```

### Save unlock flag on trigger (inside each `UNLOCK_THRESHOLDS` entry's `onUnlock`, after setting the in-memory flag)

For the fish-can-talk entry specifically:
```js
localStorage.setItem('unlock_fish_can_talk', '1');
```

### Guard one-time toast (inside `checkUnlocks`, so already-unlocked thresholds don't re-fire the toast on page load)

```js
function checkUnlocks(prev, next) {
    for (const u of UNLOCK_THRESHOLDS) {
        if (prev < u.threshold && next >= u.threshold) {
            if (!localStorage.getItem(u.key)) {   // <-- guard
                u.onUnlock();
            }
        }
    }
}
```

Each `UNLOCK_THRESHOLDS` entry will need a `key` field matching the localStorage key so the guard can check it.

---

## Scalability note

When adding future unlocks to `UNLOCK_THRESHOLDS`, add a `key` field to each entry and follow the same pattern: set the flag in `onUnlock()` and guard the check in `checkUnlocks`.
