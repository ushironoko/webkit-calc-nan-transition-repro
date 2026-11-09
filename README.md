# WebKit: CSS transition restarts every frame when the target is a calc() mixing percentage and NaN length

Reported as [WebKit Bug 330679](https://bugs.webkit.org/show_bug.cgi?id=330679).

A minimal reproduction for a WebKit bug. When a transitioned `width` or `max-width` changes to a `calc()` that mixes a percentage and a NaN length (for example `calc(50% + NaN * 1px)`), WebKit cancels and restarts the transition on every frame and never finishes it. With `?noguard`, the loop runs forever and the page's memory keeps growing.

## Live pages

- Minimal repro: <https://ushironoko.github.io/webkit-calc-nan-transition-repro/minimal.html>
- Condition matrix (60 cases): <https://ushironoko.github.io/webkit-calc-nan-transition-repro/matrix.html?autorun>

`minimal.html` stops itself after 50 `transitionrun` events. Add `?noguard` to remove that guard; the tab may then run out of memory.

## Repro

```html
<style>
  #container { width: 400px; }
  #box { width: 100px; height: 20px; background: teal; transition: all 0.2s; }
  #box.nan { width: calc(50% + var(--n) * 1px); }
</style>
<div id="container"><div id="box"></div></div>
<script>
  box.style.setProperty("--n", String(Number.NaN));
  getComputedStyle(box).width;
  requestAnimationFrame(() => box.classList.add("nan"));
</script>
```

### Expected

Per [CSS Values 4](https://drafts.csswg.org/css-values-4/#calc-ieee), a top-level calculation that produces NaN is censored to 0. The width transitions once from `100px` to `0px`, fires one `transitionrun` / `transitionstart` / `transitionend`, and stops.

### Actual (Safari 27.2, WebKit 605.1.15, macOS 27.2)

```
FAIL: transitionrun fired more than 50 times (transition restart loop)
computed width: 0px
offsetWidth: 0
running animations: 1
events: {"transitionrun":51,"transitionstart":50,"transitioncancel":50,"transitionend":0}
```

Chromium 147 passes `minimal.html` and all 60 cases in `matrix.html`.

## Matrix results in Safari 27.2

`matrix.html` crosses property (`width`, `max-width`), NaN source (`NaN` keyword, `0px / 0`, `var()`), unit (`px`, `%`, mixed), transition (none, `all`, property-specific) and direction (`100px → NaN`, `NaN → 100px`).

Every failure involves the mixed `calc(50% + NaN * 1px)`. All other NaN forms pass, including `calc(NaN * 1%)` and `calc(var(--n) * 1%)`.

| Case | Result |
|---|---|
| Static, `width: calc(50% + NaN * 1px)` | Pass. Used width is 0. |
| Static, `max-width: calc(50% + NaN * 1px)` | Layout is correct (width 0), but the computed value serializes as `calc(50% + NaN * 1px)`. Chromium returns `0%`. |
| Transition `100px → calc(50% + NaN * 1px)` | About 41 restarts in 700 ms (run/start/cancel/end = 42/42/41/0). Mid-transition computed value: `calc(47.834909% + NaN * 1px)`. |
| Transition `calc(50% + NaN * 1px) → 100px` | One extra transition (2/1/1/1). |

The behavior is the same with `transition: all` and with a property-specific transition.

## Likely cause

A calc() that mixes percentage and length cannot be resolved until layout, so it stays an expression in the computed style. Layout censors NaN correctly, but the transition logic appears to compare these unresolved expressions directly. Because NaN never equals itself, the after-change style always looks different from the running transition's end value, so the transition is cancelled and restarted on every style update.
