# An Android tablet running a custom Home Assistant dashboard

Wes’s video inspired this build: a Lenovo Tab M10 (TB-X505F), running Android 10, with a custom React
dashboard inside a small Android WebView app. Home Assistant supplies the calendar and device data, hosts the built web
files, and runs background reminders. The interface is a separate application, so its layout can be
changed freely while keeping the existing integrations.

These are application build notes. The tablet used stock Android, with no rooting or firmware
modification. They do not establish a new supported photo-frame model.

## Three layers keep display behaviour separate from the interface

```text
Android WebView app
    Full screen, keep awake, scheduled dimming, recovery, settings access
            |
Custom React + TypeScript dashboard, built with Vite
    Today, calendar, home controls, trains, car, tasks
            |
Home Assistant APIs and authenticated server-side adapters
    Calendar/device integrations, task sync, routing, notifications
```

The WebView app is deliberately small. It presents the dashboard in landscape, keeps the screen on,
persists the URL, offers a maintenance/settings exit, and retries failed loads. Network recovery and
WebView renderer recovery belong here rather than in every dashboard component. Screen dimming is a
native display setting; the web app independently switches between light and dark themes.

The web app uses React, TypeScript and Vite, with Lucide icons, bundled fonts and native SVG gauges.
There is no charting framework behind the battery, range and air-quality instruments. A few shared
components cover the circular gauges, segmented bars and tactile switches.

Home Assistant serves the compiled assets. The dashboard signs in through the normal HA flow with a
dedicated non-administrator account; tokens are not compiled into the JavaScript. Task-provider
credentials and reminder logic stay server-side. This means the development computer is not needed
for normal operation. Dashboard visibility alone is not an authorization boundary.

## Today shows decisions, with detail one tap away

The overview is sized to fit the tablet without page scrolling. Large type makes the time, next
appointment and immediate actions readable at a glance. Each subject opens a fuller page:

- Calendar combines the current/next event with coming-up events. Events with a physical location can
  show a leave-home time from a driving estimate plus an arrival buffer.
- Train pickups expand during configured collection periods. Outside those periods, the small card
  selects the next arrival whose leave-home deadline has not passed. A “Picked up” button collapses
  it for the remainder of that period. The full page has operator filters and a scrolling timetable.
- Home controls use rocker-style light and heating switches, temperature sliders, room meters and
  air-quality gauges. Switch state follows the device report, and controls disable while unavailable.
- The car page separates battery percentage, range, charging state and cable connection. It includes
  electrical readings, warning lamps, service countdowns and an estimated service date.
- Tasks show an unfinished My Day item with a Next button on Today. The card disappears when that list
  is empty. The full view has a list sidebar, completion controls and task details.

Calendar columns, train rows and task panes scroll independently, leaving their navigation and
headers in place. Theme colours, bold typography and reusable controls carry across the detail pages.
Selected components also ship as a Home Assistant custom card, using HA’s supplied `hass` object and
shadow-root styles. The same instruments can therefore appear in a normal HA dashboard.

## Background reminders do not depend on the screen being open

An authenticated adapter runs the `ms-todo` CLI beside Home Assistant for task synchronization and
completion. Its My Day selection is maintained by that CLI: it does not claim to read My Day choices
made only in Microsoft’s app. That limitation is shown in the interface.

The journey monitor compares an upcoming event’s round-trip road distance against the car’s reported
range. It estimates the missing charge and works backwards through configured off-peak windows before
departure. An amber reminder gives time to prepare; red indicates an approaching deadline or a plan
that cannot fit. The dashboard promotes the car card until charging starts or the shortfall clears.
The monitor advises; it does not change the charger’s schedule. A plugged-in cable alone does not
count as charging.

Air-quality notifications require a sustained elevated reading, with a separate recovery threshold
to avoid repeated alerts around a boundary. Reminder state persists across server restarts to avoid
sending the same notification repeatedly.

## Freshness and failure states needed as much work as the layout

Several details made a difference in actual use:

- Reading calendar events did not necessarily refresh the provider’s locally synced copy. Explicit
  integration refreshes made deleted events disappear, and a manual Refresh button gives a direct
  way to recheck the display.
- A failed request must not look like an empty calendar or a free day. Missing sensor readings stay
  unavailable rather than becoming zero. Charging checks with stale battery data cannot claim enough
  range or “all clear.”
- Driving estimates need provenance. A road-only fallback is labelled as lacking live traffic;
  current traffic is not presented as a prediction of traffic tomorrow.
- Service dates are anchored to the vehicle’s health-report timestamp plus its reported days
  remaining. Adding the countdown to today on every render would move the date forward when a
  report is stale. Service bars have labelled reference scales, not an invented percentage since
  the last service.
- Display-unit conversion happens at the presentation boundary. Miles can be shown while route and
  charging calculations retain a consistent internal unit, without converting an already-imperial
  reading twice.
- Builds use hashed assets and a version manifest. Assets are installed before switching the entry
  page; the old assets remain available for rollback. The display checks for a new version when idle
  and on resume/reconnect, avoiding a stale page after an update.

## Verification used the tablet as well as a desktop preview

The implementation was iterated on the actual tablet, including touch targets, both themes, page
height, scroll containment, and visible unavailable states. Focused tests cover time boundaries,
calendar dates, reachable-train selection, unit conversion, charging decisions and notification
deduplication. Wi-Fi reconnection and app-restart behaviour were checked separately from layout.

A longer wall-power trial remains outstanding. The build notes do not claim that a short rendering
or reconnect check proves unattended reliability. Screenshots and deployment exports are omitted
because the running display contains real household information.
