# Handover: closing a pinned tab must not delete it (Windows)

Branch: `mac-look` of TheDudeCommits/WinDiaTweaks. The last release is `v1.3.2-mac.7`, and it's installed on the PC (`dude-pc`).

## The bug

On Windows, closing a pinned (bookmarked) tab in Dia's sidebar also deletes it from the pinned list. It should act like Arc on the Mac:

- **✕ on a pinned tab** closes or unloads the tab, but it **stays pinned** in the sidebar. Clicking it again loads its pinned page.
- **– on a pinned tab that has moved to another page** goes back to its pinned page. This part already works.
- **Ctrl+W on a pinned tab** should also keep it pinned. Check what Dia does today first, but be careful: the test can delete a pinned tab.

## Cause (found)

This comes from the `arcPinnedTabs` tweak in `src/ui.cpp`, in the section that starts at `// Arc pinned tabs` (around line 5436):

- `PlacePinButton` (around 5596) shows our own ✕ / – button in a popup over the right end of the row.
- `PressPinButton` (around 5518) presses Dia's own **"Close tab"** button on the row when the ✕ is clicked.

On a pinned tab, Dia's "Close tab" **removes the pin**. The code comment says so: Dia hides that button on pinned tabs outside folders, and its context-menu Close does the same. So the ✕ deletes the bookmark.

## What the fix needs

There's a cheap way and a real way. Do the cheap one only if the real one isn't possible.

1. **Find Dia's "close but keep" path, if it has one.** Possible places:
   - An item in the pinned tab's right-click menu, such as Unload, Close, Reset or Archive.
   - A ViewModel command or method in `Dia.dll`, for example something like `UnloadTab`, `Discard` or `ResetPinned`.
   - Ctrl+W on a pinned tab, if Dia keeps the pin there (Dia on the Mac does).

   If one exists, make `PressPinButton` (and Ctrl+W, if needed) use it instead of "Close tab".
2. **If Dia has nothing,** stop the ✕ from deleting:
   - At minimum: on a pinned tab at its own page, don't press "Close tab". Instead, switch to another tab, as Arc does on close, and leave the pinned row alone.
   - Unloading the page may then need a call into Dia that hasn't been found yet.
   - Don't recreate the pin after the delete. That's fragile, and it changes the order and folders.

Also check pinned tabs **inside folders**. There Dia shows its own ✕ (`PlacePinButton` skips those rows). Find out whether that ✕ also deletes the pin.

## Investigation so far

- **`diatweaks probe FLYOUTS`** lists only the right-click menus Dia has already built. The tab rows build theirs when opened, so they didn't show up. The only menus listed were the address box menu and the extensions menu (OverflowButton).
- **A string search** of the Dia package for these words was started on the PC:
  - Back to Pinned URL
  - Unload
  - Close Pinned
  - Reset Pinned
  - Pinned Tab
  - Remove from
  - Unpin
  - Close Tab
  - Archive

  The script is `C:\Users\amirr\strs.ps1`, run over ssh. It scans every dll, which is slow, and it hadn't returned anything when the session ended. Run it again on `Dia.dll` alone, plus any `.pri` / `.xbf` files, to make it fast.
- **Dia's version** is 1.7.2.2, at `C:\Program Files\WindowsApps\TheBrowserCompany.Dia_1.7.2.2_x64__ttt1ap7aakyb4\`.
- **The Mac version** for comparison is `~/DiaTweaks/src/dylib/Patches/PinnedTabCloseButton/`. There it only clears `suppressesCloseButton`, because Dia's own close on the Mac keeps the pin.

## Working with the PC

- `ssh amirr@dude-pc` (over Tailscale) opens a Git Bash shell.
- **Install location:** `~/WinDiaTweaks/bin/`, with `diatweaks.exe`, `tap.log`, `probe.out` and `tweaks.conf`.
- **Commands that need Dia's desktop session** have to go through the scheduled-task helper, because over plain ssh `diatweaks.exe` says Dia isn't running with tweaks. That includes `probe`.

  `wdt-task.ps1` runs another script in the user's desktop session and prints its output:

  ```bash
  ssh amirr@dude-pc 'powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Users\amirr\wdt-task.ps1" -Script wdt-probe2.ps1'
  ```

  `wdt-probe2.ps1` currently runs `diatweaks.exe probe FLYOUTS` and prints `probe.out`. Edit it for other probe names. Useful ones:
  - `Name*` dumps the tree under that element.
  - `HIT:x,y` shows what's under a point.
  - `FOCUS` shows which element has keyboard focus.
- **Screenshots:** `wdt-task.ps1 -Script wdt-shot.ps1` saves `~/wdt-shot.png`. Copy it back with scp.
- **To release:**
  1. Push a tag `v1.3.2-mac.N`. The CI workflow `release.yml` builds `WinDiaTweaks.zip` for it.
  2. Then install it on the PC (this restarts Dia):

     ```bash
     ssh amirr@dude-pc 'powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Users\amirr\wdt-task.ps1" -Script wdt-update.ps1 -ScriptArgs "-Tag v1.3.2-mac.N -Sha256 <zip sha256>"'
     ```

## Ground rules from the user

- Don't test intrusively while the user is using the PC, for example clicking, moving the pointer or restarting Dia. Ask first.
- **Never test by closing a real pinned tab.** That's exactly what deletes the user's bookmarks.
- On the Mac, don't restart Dia or switch tabs without asking.
- Don't copy the user's account email from the logs anywhere.

## Still unverified from the last change (v1.3.2-mac.7)

The Windows site panel (the extensions panel) and the capsule extensions are installed. Nobody has looked at them on screen yet, and Bookmark (Ctrl+D) and the extension tiles haven't been tried.
