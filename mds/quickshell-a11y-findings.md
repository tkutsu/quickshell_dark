# Quickshell accessibility findings

Tested 2026-10-09: Quickshell 0.3.2, Qt 6.12.0, Hyprland, HDMI-A-1
(2560x1440). Branch `a11y-bar`, based on `4dce720`.

AT-SPI can expose and activate the bar and popup controls. Coordinates need
compositor offsets; AT-SPI does not identify which bar owns a popup. Keep the
wayhint provider proposal until its window mapping has been decided. The
wayhint plan was not edited.

## Changes

`shell.qml` enables `QT_LINUX_ACCESSIBILITY_ALWAYS_ON=1` through Quickshell's
[environment pragma](https://quickshell.org/docs/v0.3.0/guide/advanced/#environment).
No launcher script or global accessibility setting changed.

Named press actions cover shared ClickArea/BarItem, popup buttons and actionable
rows, menu entries, workspaces, application groups, calendar cells, music and
countdown controls. Settings controls expose names and checked state. Inputs
have names. Sliders expose a normalized 0..1 range and steps; a value proxy
routes direct AT-SPI writes through the existing `moved` callback without
turning model updates into writes.

BarItem selects one accessibility callback, giving its popup the same left
click it takes from the module's actions. Separate inherited press handlers
initially toggled the drawer twice; the regression check verifies one action.
Overlapping popup/pin hit areas are ignored. Right/middle clicks remain pointer
actions: Qt's [Accessible actions](https://doc.qt.io/qt-6/qml-qtquick-accessible.html)
have no matching mouse-button action.

## 1. Windows

The tree contains a `quickshell` application with layer-shell windows as
`frame` children. PopupWindows and overlays appear while open. Tested popup
frames have empty names/attributes and no AT-SPI parent-window relations.

Testing used a bottom Overlay bar, namespace `quickshell:a11y-test`, no
exclusive zone or backdrop, isolated state and a private session/accessibility
bus. Popup anchors were adjusted upward in the scratch copy only.

## 2. Coordinate frames

Hyprland reported the scratch bar at `(0, 1402, 2560, 38)`; AT-SPI reported
`(0, 0, 2560, 38)`. SCREEN is Qt's coordinate frame, not Hyprland's position
for these surfaces. WINDOW coordinates are local to the containing window.

| Window | AT-SPI SCREEN extents (x, y, w, h) | Position plus bar offset |
| --- | --- | --- |
| Bar | 0, 0, 2560, 38 | 0, 1402 |
| Calendar | 1107, -258, 240, 280 | 1107, 1144 |
| Display | 2193, -129, 314, 151 | 2193, 1273 |
| Audio | 1968, -178, 260, 200 | 1968, 1224 |

A calendar day reported SCREEN `(1241, -169, 28, 28)` and WINDOW
`(134, 89, 28, 28)`. Its SCREEN position includes the popup's anchor offset.
Adding the bar's compositor offset placed the calendar screenshot correctly;
the display/audio positions above are calculated using the same offset.

## 3. Popups and actions

Actual AT-SPI action 0 opened Calendar, Audio, Display and Search, and toggled
the drawer once. Calendar's next-month/current-month actions changed its month.
The existing Restart action opened the power confirmation, and Cancel left it;
no power command ran. An inert slider fixture verified AT-SPI Increase and
Value.set_current_value each emitted `moved` once, changing 0.5 to 0.55 and
then 0.8. Before the proxy fix, the setter succeeded but skipped that callback.

`hyprctl -j layers` provides the bar's namespace, PID and dimensions, but did
not list PopupWindows. On this one-bar desktop, its offset locates anchored
popups. Multiple bars/screens need an association AT-SPI did not provide here.
Triggering through AT-SPI and placing tags from a known popup anchor remains
a possible fallback. Multiple monitors and popup sliding at screen edges
were not verified.

## 4. Hover and excluded areas

BarItem and workspace/tray tooltips remain hover driven; their parent targets
have extents. With a popup open, hovering another popup control replaces it;
Music uses its title. Moving to Calendar's extents plus the bar offset
successfully browsed from Display to Calendar. The pointer was restored;
no typing was injected.

Calendar days have named extents for agenda hover. Agenda updates with Google
credentials and external tray submenu hover were not exercised on the private
bus. Submenu rows have named press actions as well as their existing hover.

Left without button annotations: Bar's pass-through dismissal observer;
launcher/settings/power padding catchers; AudioPopup's wheel area; EmailPopup's
right-click link copier; Slider's pointer drag area; WallpaperPopup's crop drag;
and MusicPopup's queue drag/click observer. Queue rows expose individual play
actions instead. Rich text links and scrollbar/drag operations were not
separately converted to buttons.

## 5. Checks and cost

- All 49 changed QML files passed `/usr/lib/qt6/bin/qmlformat` parsing. Older
  `/usr/bin` tools rejected unchanged source too.
- `python -I scripts/test-a11y.py` passed with real shared components: pointer
  parity, one inherited activation, popup routing, guards, slider steps/value
  writes/model updates, and settings toggles/radio selection.
- The existing `scripts/test-countdown-clicks.py` passed.
- `scripts/a11y-tree.py` dumped roles, names, states, actions, value ranges and
  SCREEN/WINDOW/PARENT extents. `--pid` selects a shell; `--press` requires that
  filter and one visible match. Captures and scratch copies are retained in
  ignored `.a11y-test/`.

Three interleaved startup samples per setting, process creation to
`Configuration Loaded`: off 384.1/394.6/404.4 ms; on 403.9/393.9/384.0 ms.
Medians: 394.6/393.9 ms. Four-second idle CPU samples: off 0/1/0%, on 0/0/0%.
No measurable increase in these short scratch samples; long-term live cost
remains unverified.

Qt warned that `AtSpiAdaptor::applicationInterface` does not implement
`GetApplicationBusAddress` when clients queried the tree. This also occurred
before annotation and did not stop actions. Missing MPD sockets, the absent
test display script and private portal registration produced fixture warnings
in both settings. No annotation-related QML binding/type warning was observed.

Visible captured targets had roles, names and extents inspected. Not every
new action was invoked: workspace focus, network/Bluetooth changes, ejection,
upgrades, power commands, authenticated mail/tasks, music queue edits and
weather data-dependent targets remain unverified end to end. Shared-handler
checks establish routing, not those integrations.

## Live state

The branch was merged into `main` (`83a58c5`). The desktop accessibility
socket refused connections until `at-spi-dbus-bus.service` was restarted on
2026-10-09 23:27; a bar started before that logged `Error in contacting
registry` and stayed off the bus, so the bar needs a restart after the bus
does.

Rechecked live afterwards (one top bar at `(0, 0)` on HDMI-A-1): the bar frame
reported SCREEN `(0, 0, 2560, 38)`, matching Hyprland's layer, so SCREEN
extents are global coordinates on this desktop. Action 0 on Calendar opened
it; its frame reported SCREEN `(1107, 34, 240, 366)`, directly under the bar,
with named month and day buttons, and a second press closed it. Two things
for a consumer of the tree: controls that are collapsed or hidden (the music
pill's buttons, an idle countdown) stay in the tree without `showing` or
`visible`, so filter on `showing`; and each button lists `Press` twice (the
Button role's own and `onPressAction`'s), both doing the same thing.

## 6. Hover and scroll areas (2026-10-10)

A `ClickArea` now stays in the tree when it only hovers (a module with a
tooltip, `hovers`) or only scrolls (`scrolls`), as role `label` rather than
`button`; one with nothing to press, hover or scroll is still ignored.
Every wheel area answers Qt's `Scroll Up` and `Scroll Down` actions, checked
live over AT-SPI: the workspaces strip (now named "Workspaces"), clock,
timer, audio, wallpaper, display, keyboard layout and music modules, tray
icons (one notch to the app), the weather popup's graphs (a day each way),
and the popups' lists (notifications, music queue, mail, wallpapers,
launcher results and preview, settings), half a view per action.

Labels still list `Press`, because ClickArea's handler is always connected;
it does nothing on them. Tell them apart from buttons by role.
