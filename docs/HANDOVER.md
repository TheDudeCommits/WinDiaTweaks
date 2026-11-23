# Handover: Windows Dia to match the Mac (Arc look)

Branch: `mac-look` of TheDudeCommits/WinDiaTweaks. Last release **`v1.3.2-mac.8`** (everything under "Done"),
which the PC (`dude-pc`) runs. Dia on the PC is now **1.7.3.2**.

The reference is the MacBook Air's Dia with DiaTweaks (`~/DiaTweaks`, its settings in
`~/Library/Preferences/com.diapatches.plist`), which is styled after Arc. Screen capture over ssh is blocked by
macOS privacy settings on both Macs, so the Mac source and settings were the reference, not screenshots.

## Done (tested on the PC with throwaway tabs, released in v1.3.2-mac.8)

- **Pinned ✕ keeps the pin.** Found by testing on throwaway pins: Dia's row button "Close tab" removes a pinned
  tab (in the list, and in groups where Dia shows it itself), but Dia's right-click **Close** (and Ctrl+W)
  closes the tab and keeps it pinned, as Arc does. There is no Unload item. The ✕ now right-clicks the row
  (pointer goes straight back), hides the menu as it's added, and presses its Close. Our ✕ also covers Dia's
  own ✕ on pinned tabs in groups. Grouped pinned tabs have no "Unpin" in their menu; "Edit pinned page" marks
  them as pinned. Pinned tiles (the dock) keep their pin on Close too.
- **The ✕ never showed on Dia 1.7.3**: the row now marks pointer events handled; hover handlers take handled
  events.
- **Collapse pinned tabs** (`collapsePinned`): the Mac's "Pinned" line above the pinned tabs and folders; click
  folds them (rows below slide), remembered per profile page.
- **Keyboard** (Mac's DiaTweaks shortcuts, Ctrl for ⌘): Ctrl+D pin/unpin (Dia's Ctrl+Shift+D; Dia's Ctrl+D
  bookmark moves to the site panel), Ctrl+1–9 profiles (Dia's Alt+1–9) with Alt+1–9 taking Dia's Ctrl+1–9,
  Ctrl+Z right after closing a tab reopens it, Ctrl+Shift+X extensions menu. In the Ctrl+T key hook; the hook
  lets by only keys it sent itself (`kOwnKeys` in dwExtraInfo).
- **Capsule buttons on hover**: Dia's capsule buttons (pinned extensions, copy link, site ⋯) are moved over the
  pill from the page's toolbar, but the page's rounded Border (radius 7) clips them, so they never showed.
  Look-alikes in the pill now press Dia's buttons. The site label also kept its narrow width after the first
  hover (fixed).
- **Loading in the pill**: Dia's loading indicator is clipped the same way; the pill shows a live visual-surface
  copy of it (the Mac's Show Loading Progress).
- `cleanSearchBox` turned on (the Mac's Compact Command Bar).

## Not done

- **Open new tabs next to current**: Dia appends new tabs at the end; moving one needs a synthetic drag.
- **Peek from pinned tabs**: Dia already opens a pinned tab's cross-site link in a new tab; turning that into a
  Peek needs a reliable "the active tab is pinned" signal (the selected tile only has a brighter fill).
- **Close tabs from the Ctrl+Tab switcher**: the switcher isn't a reachable XAML element or popup.
- **Little Dia**: Edge is the default browser on the PC, so links from other apps don't reach Dia.
- Drag window from page, Open sidebar instantly, Keep folders open: not looked at.
- **Open question for the user**: the Mac shows **−** (not ✕) on pinned tabs with a page open, and that −
  closes the page and keeps the pin. Windows shows ✕ there (as this handover asked) and − only for "back to
  pinned page".

## Working on the PC

The Claude app now runs on the PC itself. Build locally (`build.cmd` works with VS 2022 Community):
`~/Claude/WinDiaTweaks/dev-build.sh` builds; Dia must be closed to swap the DLL, which
`~/wdt-swap.ps1` does (close Dia, copy the build into `~/WinDiaTweaks/bin`, start DiaLauncher). After adding a
tweak to `tweaks.conf`, merge it into the installed conf:
`~/WinDiaTweaks/bin/diatweaks.exe merge-conf <repo>\tweaks.conf <home>\WinDiaTweaks\bin\tweaks.conf`.

Everything that touches Dia's desktop runs through `~/wdt-task.ps1 -Script <script>` (a scheduled task in the
desktop session). Each task launch steals focus, which closes Dia's popups and the Ctrl+T bar, so do keys,
clicks and screenshots in one run: `~/wdt-seq.ps1 -Steps "focus;keys:^t;sleep:800;shot:C:\path.png;..."`
(steps: focus, keys, type, sleep, move, click, rclick, wheel, kdown, kup, shot, probe, wins).

- Hover only registers with **absolute** mouse moves (`wdt-seq`'s move/click glide in from the right).
- SendKeys input is injected; the key hook handles it like real keys.
- Screen is 3840×2160 at 150%: screen pixels = DIPs × 1.5 (probes print DIPs).
- Probes (`~/wdt-probe.ps1 -Names "..."`): `ROWS`, `TILES`/`LIST` (switch targets), `ROWRMENU:i` (right-click a
  row and list its menu), `ROWRITEM:i:Text_with_underscores` (press a menu item), `ROWBTN:i:Close_tab`,
  `MENU`, `PINSTATE`, `TABLIST`, `UP:Name`, `NAMES:text`, `HIT:x,y`, `FOCUS`, `Name*`. Async menu output goes
  to `probe2.out`.
- Test pins: Ctrl+T a throwaway URL, then drag its row above the divider (`~/wdt-drag.ps1`), or Ctrl+D (→ dock
  tile). Delete with `ROWBTN:i:Close_tab`. Check `…\User Data\Default\Bookmarks` for stray test bookmarks.

## Ground rules from the user

- Don't test intrusively while the user is using the PC; check idle time first (`~/wdt-idle.ps1`).
- Never test by closing a real pinned tab.
- On the Mac, don't restart Dia or switch tabs without asking.
- Don't copy the user's account email from the logs anywhere.
- Push and release only when asked. Release: push a lightweight tag `v1.3.2-mac.N` (CI builds `WinDiaTweaks.zip`
  and writes the notes from the commit subjects), then `~/wdt-update.ps1 -Tag … -Sha256 …` through `wdt-task.ps1`
  (it closes and reopens Dia, so check idle time first).
