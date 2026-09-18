# VibrateAuto

**Android app that switches your phone to vibrate or silent at synagogues and your own places/times, then restores your ringer when you leave.**

![Platform](https://img.shields.io/badge/platform-Android%208.0%2B-3DDC84)
![Framework](https://img.shields.io/badge/.NET-10%20MAUI-512BD4)
![Language](https://img.shields.io/badge/language-C%23-239120)
![Source](https://img.shields.io/badge/source-private-lightgrey)

| Your rules | Creating a rule | Guided setup |
|---|---|---|
| ![Rules list](screenshots/main-rules.png) | ![Rule editor](screenshots/rule-editor.png) | ![Setup checklist](screenshots/setup.png) |

---

## The problem

A phone ringing during prayer services is an avoidable embarrassment that happens to everyone. The usual answers do not work well:

- **Remembering to do it manually** fails exactly when you are distracted, which is most of the time.
- **Generic automation apps** are built for people who enjoy configuring automation apps. They ask you to define triggers, conditions and actions before anything happens, which most people find annoying rather than convenient.
- **Calendar-based silencing** does not know that you arrived early, stayed late, or went to a different service than usual.

The people I built this for are often older, not especially technical, and in many cases using a deliberately simplified handset with a small, low-specification screen. A solution for them has to work without being configured, and has to be impossible to misunderstand.

## What it does

VibrateAuto already knows where synagogues are, and lets you add your own places for everywhere else.

- **Automatic synagogue detection.** On first run it pulls the nearest synagogues from a public directory and quietly fences all of them. No setup, no typing.
- **Your own rules too.** Add a place, a time window, or both. A rule can cover an office, a library, or a weekly class.
- **Vibrate or full silent**, chosen per rule.
- **Restores your ringer when you leave**, which matters more than the silencing does.
- **Switch off individual synagogues**, for the one across the road from your house.
- **Pause everything** with one control when you want the phone to behave normally.
- **Tells you when it is not working.** If a permission is missing, the app says so on the main screen rather than failing silently.

## How it works

The short version: the operating system watches for you crossing a boundary, wakes the app, and the app decides what your ringer should be.

```
Geofence crossing ─┐
Scheduled alarm ───┤
Device reboot ─────┼──► RingerEvaluationService ──► what should the ringer be right now?
App opened ────────┤         (single decision point)
Movement detected ─┘
```

Every trigger funnels into one evaluation method. That was a deliberate choice: an earlier design had several paths that could each change the ringer, and it was possible to re-register geofences without re-checking the ringer, which left phones stuck on vibrate after a reboot.

Full detail, including the geofence strategy and the battery model, is in **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**.

## Engineering challenges worth mentioning

### Staying inside Android's 100-geofence limit

Android rejects the entire registration request if an app asks for more than 100 geofences. A dense neighbourhood has far more synagogues than that. Querying the directory for central Lakewood, New Jersey returns **414 synagogues within 20 miles**, and the nearest 80 of them sit inside a **1.1 mile radius**.

The app registers the nearest 80 plus one large "boundary" fence, and re-fetches the list only when you leave that boundary. The boundary is sized to the set actually fetched rather than being a fixed distance, so in a dense town it is about a mile and in a sparse one it is fifteen. An earlier fixed 15-mile boundary meant that driving three miles across town left the user with no coverage at all and no refresh.

When the total still exceeds the platform cap, synagogues are trimmed first and the user's own rules are always kept.

### Battery, without sacrificing responsiveness

Holding a GPS lock all day is not acceptable for an app that runs constantly. While a rule is active the app runs a foreground service that:

- polls at **30 second** intervals at high accuracy while you are moving,
- drops to **3 minute** intervals at balanced power once you have been stationary for three minutes,
- and arms Android's **significant motion** hardware sensor while slow, so standing up to leave is noticed within a second or two without any polling at all.

The chip does the watching instead of the app. An hour inside costs roughly three minutes of GPS.

### Never leaving the phone silently broken

The worst possible outcome for this app is not failing to silence the phone. It is silencing the phone and never giving it back. Four independent safeguards exist for that:

1. **An accuracy gate.** Any position fix worse than 40 metres is treated as a guess and ignored, rather than acted on.
2. **A hysteresis band.** Entry is confirmed inside the rule's radius, exit only once you are 60 feet beyond it, so a jittering fix cannot make the ringer flap on and off.
3. **A stale-override watchdog.** If the app has held the ringer for three hours without a single confirming position fix, it hands the ringer back on its own.
4. **A manual escape hatch.** The persistent notification carries a Restore ringer button that does not depend on any geofence or alarm arriving.

Because that state has to be obvious rather than discovered, pausing is a full-width banner rather than a changed button label:

![Paused state](screenshots/paused.png)

### Surviving Android's background restrictions

Geofences do not survive a reboot, and Android will shut down an app that has been idle for a few days. The app re-registers everything on boot, runs a daily maintenance alarm as a health check, and walks the user through a battery-optimisation exemption during setup. Because geofencing and fused location both come from Google Play Services, the setup checklist also verifies Play Services is present and usable, which a stripped-down handset cannot be assumed to have.

### Designing for a 320 dp screen

- Every screen is laid out for **320 × 640 dp**, with 360 dp treated as the bonus rather than the baseline.
- Everything survives the Android **1.3× font scale** without clipping, because heights are minimums and text wraps rather than truncating.
- Every touch target is at least **48 dp**.
- Editing a rule is a **tap on the card**, not a swipe. Swipe was undiscoverable for this audience, and a hidden gesture is a feature that does not exist.
- All icons are drawn as **vector path geometry** rather than emoji or an icon font, because a stripped-down ROM may render either as empty boxes.
- Rows disappear rather than leaving orphans. A time-only rule shows no location line at all, instead of an icon with nothing beside it.

## The code

The source is private, but these four excerpts show the kind of decisions the app is made of. Each one exists because something went wrong first.

### Hysteresis, so a jittering fix cannot flap the ringer

```csharp
double enterMiles = enterFeet / 5280.0;
double exitMiles = (enterFeet + ConfirmExitMarginFeet) / 5280.0;

double distanceMiles = Location.CalculateDistance(
    rule.Latitude, rule.Longitude, location, DistanceUnits.Miles);

if (distanceMiles <= enterMiles) return true;   // close enough to be inside
if (distanceMiles >= exitMiles) return false;   // far enough to have left

// In the narrow band between the two. Keep whatever is already applied.
return isCurrentlyActive;
```

### Sizing the resync boundary to the data instead of guessing

```csharp
private static double FitBoundaryRadiusMiles(List<VibrationRule> shulRules, Location center)
{
    // Fewer results than the cap means the fetch already covered everything in range,
    // so a tighter circle would only trigger resyncs that return the same list.
    if (shulRules.Count < MaxShulGeofences)
    {
        return MaxBoundaryRadiusMiles;
    }

    double farthestMiles = 0;

    foreach (var rule in shulRules)
    {
        double miles = Location.CalculateDistance(
            rule.Latitude, rule.Longitude, center, DistanceUnits.Miles);

        if (miles > farthestMiles)
        {
            farthestMiles = miles;
        }
    }

    if (farthestMiles < MinBoundaryRadiusMiles) return MinBoundaryRadiusMiles;
    if (farthestMiles > MaxBoundaryRadiusMiles) return MaxBoundaryRadiusMiles;

    return farthestMiles;
}
```

### The watchdog that guarantees you get your ringer back

```csharp
public static void RestoreIfOverrideIsStale(Context context)
{
    if (!AudioManagerService.IsOverrideActive()) return;

    DateTime lastConfirmed = AudioManagerService.LastConfirmedInsideUtc();

    if (DateTime.UtcNow - lastConfirmed < MaxTimeWithoutConfirmation) return;

    AppLogger.Error("Evaluate",
        "Ringer override held with no confirming location fix since "
        + lastConfirmed.ToString("u") + " - restoring the ringer");

    AudioManagerService.ApplyRuleRingerTarget(context, null);
    ActiveRingerWatchService.Stop(context);
}
```

### Adaptive polling measured from a fixed anchor

Comparing each fix to the previous one meant indoor GPS drift looked like movement, so the service never slowed down and never got cheap.

```csharp
// True displacement from the fixed anchor, not the last-ping delta.
float displacementMeters = _anchorLocation.DistanceTo(location);

if (displacementMeters < StationaryThresholdMeters)
{
    // Still near the anchor. GPS jitter does not move the anchor or reset the clock.
    if (_isFastPolling &&
        DateTime.UtcNow - _stationaryStartTime >= StationaryBeforeBackoff)
    {
        _isFastPolling = false;
        RegisterLocationRequest(SlowIntervalMs);   // 30 s -> 3 min, balanced power
    }
}
else
{
    // Real movement. Re-anchor and go back to fast, high-accuracy polling.
    _anchorLocation = location;
    _stationaryStartTime = DateTime.UtcNow;

    if (!_isFastPolling)
    {
        _isFastPolling = true;
        RegisterLocationRequest(FastIntervalMs);
    }
}
```

## Tech stack

| Area | Choice |
|------|--------|
| Framework | .NET 10 MAUI, C# |
| Target | Android only, minimum Android 8.0 (API 26), targeting API 36 |
| Location | Google Play Services geofencing and fused location |
| Maps | Google Maps SDK for Android, Google Places for address search |
| Synagogue data | Public synagogue directory API |
| Storage | Local JSON plus platform preferences, no backend, no accounts |
| UI | XAML with a hand-built design system, dark theme only |

**Scale:** 52 C# files totalling roughly 4,900 lines, 10 XAML files totalling roughly 2,250 lines, across 5 screens, 13 shared services and 16 Android platform services and broadcast receivers.

No user account, no analytics, and no server. Location never leaves the device except as a coordinate pair sent to the directory to ask what is nearby.

## Design system

The interface is dark-only and built on a fixed palette and type scale rather than ad-hoc values. Details, including the full token list and the spacing rhythm, are in **[docs/DESIGN.md](docs/DESIGN.md)**.

![Settings](screenshots/settings.png)

## Status

The app is feature-complete and running on a physical device. It has not been released to the Play Store.

**Working:** synagogue sync and geofencing, custom rules with time and location, ringer switching and restore, the full setup and permissions flow, and the complete redesigned interface.

**Roadmap:** field testing across several weeks and devices, distance display on the nearby synagogues list, loading indicators during network calls, and a Hebrew localisation, which the layout was already built to support.

## Source code

The full source is kept private. The excerpts above are representative; I am happy to walk through the codebase or share access for a technical review — please get in touch.
**[Connect with me on LinkedIn](https://www.linkedin.com/in/shia-grosinger/)**
---

*Built by Shia Grosinger.*
