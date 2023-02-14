# urob's zmk-config

This is my personal [ZMK firmware](https://github.com/zmkfirmware/zmk/) configuration.
It consists of a 34-keys base layout that is re-used for various boards, including my
Corneish Zen and an Advantage 360 pro.

## *Key* features

- clean keymap + unicode setup using helper macros from
  [zmk-nodefree-config](https://github.com/urob/zmk-nodefree-config)
- the base keymap and combo setup are independent of the physical location of
  keys and are re-used for multiple keyboards. The configuration is fit onto
  larger boards by padding it via a modular structure of "extra keys"
- ["timer-less" homerow mods](#timeless-homerow-mods) 
- combos replacing the symbol layer
- a smart-layer implementation for ZMK that automatically toggles the numbers
  layer
- long-pressing the arrow-cluster yields home, end, begin/end of document, and
  fwd/bwd-delete words
- sticky shift on right thumb, double-tap (or shift + tap)[^1] activates
  caps-word
- <kbd>shift</kbd> + <kbd>,</kbd> morphs into <kbd>;</kbd>  and
  <kbd>shift</kbd> + <kbd>.</kbd> morphs into <kbd>;</kbd> (freeing up the
  right pinky for <kbd>? / !</kbd>)
- <kbd>shift</kbd> + <kbd>ctrl</kbd> + <kbd>,</kbd> morphs into <kbd><</kbd>
  and <kbd>shift</kbd> + <kbd>ctrl</kbd> + <kbd>.</kbd> morphs into
  <kbd>></kbd>
- <kbd>shift</kbd> + <kbd>space</kbd> morphs into <kbd>dot</kbd> →
  <kbd>space</kbd> → <kbd>sticky-shift</kbd>
- "Greek" layer for mathematical typesetting (activated via sticky-layer combo)
- modified Github Actions workflow that recognizes git-submodules
- automated
  [build-scripts](https://github.com/urob/zmk-config/tree/main/scripts#readme)
  for local and Docker-based building (independently of VS Code)

![](img/keymap.png)

## Timeless homerow mods

[Homerow mods](https://precondition.github.io/home-row-mods) (aka "HRMs") can
be a game changer -- at least in theory. In practice, they require some finicky
timing: In its most naive implementation, in order to produce a "mod", they
must be held *longer* than `tapping-term-ms`. In order to produce a "tap", they
must be held *less* than `tapping-term-ms`. This requires very consistent
typing speeds that, alas, I do not possess. Hence my quest for a "timer-less"
HRM setup.[^2]

After months of tweaking, I eventually ended up with a HRM setup that is
essentially timer-less, resulting in virtually no misfires. Yet it provides a
fluent typing experience with mostly no delays.

Let's suppose for a moment we set `tapping-term-ms` to something ridiculously
large, say 5 seconds. This makes the configuration timer-less of sorts. But it
has two problems: (1) To activate a mod we will have to hold the HRM keys for
what feels like eternity. (2) During regular typing, there are delays between
the press of a key and the time it appears on the screen.[^3] Enter two of
ZMK's best configuration options:
* To address the first problem, I use ZMK's `balanced` flavor, which produces a
  "hold" if another key is both pressed and released within the tapping-term.
  Because that is exactly what I normally do with HRMs, there is virtually
  never a need to wait past my long tapping term (see below for two
  exceptions).
* To address the typing delay, I use ZMK's `global-quick-tap` property, which
  immediately resolves a HRM as "tap" when it is pressed shortly *after*
  another key has been tapped. This all but completely eliminates the delay.

This is great but there are still a few rough edges:

* When rolling keys, I sometimes unintentionally end up with "nested" key
  sequences: `key 1` down, `key 2` down and up, `key 1` up. Because of the
  `balanced` flavor, this would falsely register `key 1` as a mod. As a remedy,
  I use ZMK's "positional hold-tap" feature to force HRMs to always resolve as
  "tap" when the *next* key is on the same side of the keyboard. Problem
  solved.
* ... or at least almost. The official ZMK version for positional-hold-taps
  performs the positional check when the next key is *pressed*. This is not
  ideal, because it prevents combining multiple modifiers on the same hand. To
  fix this, I use a small patch that delays the positional-hold-tap decision
  until the next key's *release* ([PR
  #1423](https://github.com/zmkfirmware/zmk/pull/1423)). With the patch,
  multiple mods can be combined when held, while I still get the benefit from
  positional-hold-taps when keys are tapped.
* So far, nothing of the configuration depends on the duration of
  `tapping-term-ms`. In practice, there are two reasons why I don't set it to
  infinity:
    1. Sometimes, in rare circumstances, I want to combine a mod with a
    alpha-key *on the same hand* (e.g., when using the mouse with the other
    hand). My positional hold-tap configuration prevents this *within* the
    tapping term. By setting the tapping term to something large but not crazy
    large (I use 280ms), I can still use same-hand `mod` + `alpha` shortcuts by
    holding the mod for just a little while before tapping the alpha-key.
    2. Sometimes, I want to press a modifier without another key (e.g., on
    Windows, tapping `Win` opens the search menu). Because the `balanced`
    flavour only kicks in when another key is pressed, this also requires
    waiting past `tapping-term-ms`.
* Finally, it is worth noting that this setup works best in combination with a
  dedicated shift for capitalization during normal typing (I am a big fan of
  sticky-shift on a home-thumb). This is because shifting alphas is the
  one scenario where pressing a mod may conflict with `global-quick-tap`, which
  may result in false negatives when typing fast.

Here's my configuration (I use a bunch of [helper
macros](https://github.com/urob/zmk-nodefree-config) to simplify the syntax, but they
are not necessary):

```C++
/* use helper macros to define left and right hand keys */
#include "../zmk-nodefree-config/keypos_def/keypos_36keys.h"                // keyposition helpers
#define KEYS_L LT0 LT1 LT2 LT3 LT4 LM0 LM1 LM2 LM3 LM4 LB0 LB1 LB2 LB3 LB4  // left-hand keys
#define KEYS_R RT0 RT1 RT2 RT3 RT4 RM0 RM1 RM2 RM3 RM4 RB0 RB1 RB2 RB3 RB4  // right-hand keys
#define THUMBS LH2 LH1 LH0 RH0 RH1 RH2                                      // thumb keys

/* left-hand HRMs */
ZMK_BEHAVIOR(hml, hold_tap,
    flavor = "balanced";
    tapping-term-ms = <280>;
    quick-tap-ms = <175>;                // repeat on tap-into-hold
    global-quick-tap-ms = <150>;         // requires PR #1387
    bindings = <&kp>, <&kp>;
    hold-trigger-key-positions = <KEYS_R THUMBS>;
    hold-trigger-on-release;             // requires PR #1423
)

/* right-hand HRMs */
ZMK_BEHAVIOR(hmr, hold_tap,
    flavor = "balanced";
    tapping-term-ms = <280>;
    quick-tap-ms = <175>;                // repeat on tap-into-hold
    global-quick-tap-ms = <150>;         // requires PR #1387
    bindings = <&kp>, <&kp>;
    hold-trigger-key-positions = <KEYS_L THUMBS>;
    hold-trigger-on-release;             // requires PR #1423
)
```

Final note: the config above uses syntax introduced in [PR
#1387](https://github.com/zmkfirmware/zmk/pull/1387), which decouples the
`quick-tap-ms` timeout from the `global-quick-tap-ms` timeout. Without the PR,
one can replace `global-quick-tap-ms = <150>` with `global-quick-tap` for a
similar effect (`global-quick-tap` will use the regular `quick-tap-ms` timeout
in this case).

My personal [ZMK fork](https://github.com/urob/zmk) includes both the
global-quick-tap-ms PR and the hold-trigger-on-release PR (along with a few
other PRs). If you prefer to maintain your own fork with a custom selection of
PRs, you might find this [ZMK-centric introduction to
Git](https://gist.github.com/urob/68a1e206b2356a01b876ed02d3f542c7) helpful.


## Combo setup

My layout makes heavy use of combos. Thanks to `global-quick-tap` for combos
(introduced in above mentioned PR #1387), combo misfires are rare, even when
rolling keys. Most of my combos are bind to symbols, replacing the usual
symbols layer seen on many sub-40 keyboard layouts. The combos are designed so
as to put the most used symbols in easy-to-access locations while also making
them easy to remember. Specifically:

- the top vertical-combo row matches the symbols on a standard numbers row
  (except `+` and `&` being swapped)
- the bottom vertical-combo row is symmetric to the top row (subscript `_`
  aligns with superscript `^`; minus `-` aligns with `+`; division `/` aligns
  with multiplication `*`; logical-or `|` aligns with logical-and `&`)
- parenthesis, braces, brackets are set up symmetrically as horizontal combos
- cut (on `X + D`), copy, and paste are on the left side for one-handed mouse
  use
- `L + Y` activates Greek layer for the next key press, `L + U + Y` activates the shifted
  Greek layer the next key
- `tap`, `esc`, `enter` are on horizontal combos

## Experimental changes

- I recently reduced my core layout to 34 keys. Backspace and Delete are now on
  my Navigation-layer. To make room for these keys, I have added hold-taps to
  the arrow cluster, which now double as Home/End and Beginning/End of
  document. I really like the new navigation cluster and will likely keep it in
  one way or another
- Inspired by Jonas Hietala's
  [Numword](https://www.jonashietala.se/blog/2021/06/03/the-t-34-keyboard-layout/#where-are-the-digits)
  for QMK, I implemented my own version of [Smart-layers for
    ZMK](https://github.com/zmkfirmware/zmk/pull/1451). It is triggered via a
    single tap on my Num-key (holding the key will activate the num layer as
    usual without triggering Numword). Similar to Capsword, Numword continues
    to be activated as long as I type numbers, and deactivates automatically on
    any other keypress. I found that I use Numword for most of my numbers
    typing. For single digits, it effectively is a sticky-layer, but
    importantly I can also use it for multiple digits. The only case where it
    doesn't deactivate automatically is where immediately after a digit I would
    type any of the letters on which my numpad is located (WFPRSTXCD), which is
    rare, but does happen. For these cases I have a CANCEL key on my Nav layer
    that cancels both Numword and Capsword.
- Since the switch to 34 keys, I freed up the tap-position on my left-most
  thumb key. For now I added a secondary Bspc, but I am still searching for a
  better use. (I tried adding Repeat here but I found that it doesn't work well
  adjacent to space, which requires to much lateral thumb-movements)

## Issues and workarounds

Since I switched from QMK to ZMK I have been very impressed with how easy it is
to set up relatively complex layouts in ZMK. For the most parts I don't miss
any functionality (to the contrary, I found that ZMK supports many features
natively that would require complex user-space implementations in QMK). Below
are a few remaining issues:

- ZMK does not yet support tap-only combos
  ([#544](https://github.com/zmkfirmware/zmk/issues/544)). Workaround: pause
  briefly when chording multiple HRMs together on positions that otherwise would trigger
  a combo.
- OS sleep is not yet implemented ([#1077](https://github.com/zmkfirmware/zmk/issues/1077)).
  Workaround: use sleep-macro instead.
- `&bootloader` doesn't work with Planck_rev6
  ([#1086](https://github.com/zmkfirmware/zmk/issues/1086)). Workaround: Manually press
  reset-button.

[^1]: Really what's happening is that `Shift` + my right home-thumb morph into
  caps-word. This gives me two separate ways of activating it: (1) Holding the
  homerow-mod shift on my left index-finger and then pressing my right home-thumb, which
  is my new preferred way. Or, (2) double-tapping the right home-thumb, which also works
  because the first tap yields sticky-shift, activating the mod-morph upon the second
  tap.

[^2]: I call it "timer-less", because the large tapping-term makes the behavior
  insensitive to the precise timings. One may say that there is still the
  `global-quick-tap` timeout. However, with both a large tapping-term and
  positional-hold-taps, the behavior is *not* actually sensitive to the
  `global-quick-tap` timing: All it does is reduce the delay in typing; i.e., variations
  in typing speed won't affect *what* is being typed but merly *how fast* it appears on
  the screen.

[^3]: The delay is determined by how quickly a key is released and is not
  directly related to the tapping-term. But regardless of its length, most
  people still find it noticable and disruptive.

