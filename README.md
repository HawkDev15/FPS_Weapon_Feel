# FPS Weapon Feel

Procedural FPS weapon motion plugin for Unreal Engine 5. Adds the small-but-essential layer of life to first-person weapons: rotation lag, movement tilt, breathing, walk bob and per-shot recoil — all framerate-independent and driven by a single component.

Includes a custom 2D Recoil Pattern asset and editor for authoring weapon spray curves visually.

---

## Features

- **Rotation lag** — weapon trails behind camera turns, magnitude clamped per axis.
- **Movement tilt & lag** — body-aware tilt/offset from local velocity.
- **Breathing** — idle sway that fades out when the character moves.
- **Walk bob** — ground-locked sinusoidal bob proportional to speed.
- **Per-shot recoil** — view kick (direct ControlRotation, sensitivity-invariant) + mesh kick (offset + tilt) with exponential recovery.
- **Recoil Pattern asset** — author 2D point curves; the view follows them on consecutive shots, falling back to randomized kick after the pattern is consumed.
- **Framerate-independent** — all interpolation uses true exponential smoothing; rotation lag is normalized to angular velocity. Identical trajectories at 30, 60, 144 FPS.
- **Network-safe** — Tick and FireShot are no-ops on remote proxies and the server.

---

## Requirements

- Unreal Engine **5.3 / 5.4 / 5.5 / 5.6 / 5.7**
- C++ project (or Blueprint-only project — the component is fully BlueprintCallable).

---

## Quick Start

### 1. Attach the component

In your `ACharacter` Blueprint or C++ class, build the following hierarchy under the camera:

```
Mesh1P (Character)
└── Camera
    └── FPSWeaponFeelComponent       ← this plugin
        └── WeaponPivot (SceneComp)
            └── FP_Mesh (SkeletalMesh)
```

The component **must** be a child of the camera. The FPS arms/weapon mesh goes underneath.

### 2. Tune the per-character feel

Select the component and adjust in Details:

- **Rotation / Movement / Breathing / Walk** sections — per-character motion. Set once per character, leave alone.
- **Intensity** — global multiplier (0–1). Lower this while aiming down sights for tighter feel.
- **GlobalRecoilScale** — master recoil multiplier.

### 3. Initialize the recoil profile on weapon equip

Once per weapon equip (or whenever the active profile changes — e.g. firemode switch), cache the per-weapon profile:

```cpp
FRecoilProfile Profile;
Profile.Strength = 1.f;
Profile.ViewPitchKick = 0.4f;        // degrees up per shot
Profile.ViewYawSpread = 0.375f;      // degrees random ± horizontal
Profile.BackOffset = 7.f;
Profile.UpOffset = 1.f;
Profile.SideOffset = 0.5f;
Profile.MeshRecoverySpeed = 12.f;
Profile.RecoilPattern = MyAKPattern; // optional URecoilPattern asset

FeelComponent->InitializeRecoilData(Profile);
```

### 4. Fire shots from your weapon

On every shot:

```cpp
FeelComponent->FireShot();
```

When the player releases fire (end of burst):

```cpp
FeelComponent->EndFire();
```

That's it. The component handles everything else internally.

---

## API Reference

### `UFPSWeaponFeelComponent`

| Function | Description |
|---|---|
| `InitializeRecoilData(const FRecoilProfile& Profile)` | Cache the per-weapon recoil profile. Call once on weapon equip / profile change. |
| `FireShot()` | Apply one recoil kick. Call once per bullet from your weapon code. Uses the cached profile. |
| `EndFire()` | Signal end of a burst. Resets pattern shot index. |
| `ResetState()` | Hard-resets all accumulated motion. Call on respawn or teleport. |

### `FRecoilProfile` (per-weapon, passed into `InitializeRecoilData`)

| Field | Default | Meaning |
|---|---|---|
| `Strength` | 1.0 | Per-shot multiplier (use to scale ADS vs hip-fire). |
| `MeshRecoverySpeed` | 12.0 | How fast mesh kick recovers (1/sec). |
| `ViewPitchKick` | 0.4 | Direct pitch in **degrees** added to ControlRotation. Positive = upward. |
| `ViewYawSpread` | 0.375 | Random yaw spread in degrees (±). Used when no pattern is active. |
| `RecoilPattern` | null | Optional `URecoilPattern` asset. Each `(X=Yaw, Y=Pitch)` point in degrees. |
| `PatternResetDelay` | 0.4 | Seconds without firing before pattern index resets. |
| `BackOffset` / `UpOffset` / `SideOffset` | 7 / 1 / 0.5 | Mesh translation kick. |
| `MeshPitchKick` / `MeshYawKick` / `MeshRollKick` | 0 / 0 / 0 | Mesh rotation kick (rolled randomly within ±). Defaults disabled — translation-only kick. Increase if you want the gun to twist. |

---

## Recoil Pattern Asset

`URecoilPattern` is a 2D point list authored in a custom editor (open via Asset Editor on the asset).

- **Points are absolute view positions per shot**, relative to `Points[0]` (burst origin).
- Positive X = view moves right. Positive Y = view moves up.
- The component applies the **delta** between consecutive points each shot, so the view follows the authored curve.
- After the last point is consumed, recoil falls back to `ViewPitchKick + ViewYawSpread` (still framerate-independent).
- A pattern is **per-weapon** — store it in your weapon data and pass via `FRecoilProfile.RecoilPattern`.

---

## Tips

- **ADS:** set `Intensity = 0.3` and `GlobalRecoilScale = 0.3` while aiming. The component smoothly interpolates intensity via `IntensityInterpSpeed`.
- **Respawn:** call `ResetState()` to avoid first-frame motion spikes from accumulated state.
- **Multiple weapons:** pass a different `FRecoilProfile` per weapon. The component caches the last profile to drive Tick-time recovery, so weapon swaps just work.
- **No view kick when you want pure mesh feedback:** set `ViewPitchKick = 0` and `ViewYawSpread = 0` — the component skips the ControlRotation write entirely.

---

## Notes

- Recoil is applied directly via `PlayerController->SetControlRotation()` — **independent of mouse sensitivity, `InputPitchScale` and `bInvertMouse`**. The same `ViewPitchKick` value produces the same on-screen kick for every player.
- All interpolation uses true exponential smoothing (`1 - exp(-Speed * DeltaTime)`), not Unreal's linear `*InterpTo` approximation. Recovery curves are identical at any frame rate.
- Rotation lag normalizes input to angular velocity referenced to 60 FPS, so the lag amount is frame-rate independent.

---

## Sample Content (Free Download)

A ready-to-play sample is available as a **free separate download** — FPS character with the component pre-wired, weapon mesh + projectile, GameMode, input setup, prototyping map, and an example `Recoil Pattern` asset.

**Download:** https://drive.google.com/file/d/1Pgr2F6JmKzcF0SEmsLnlvIucQ8V_fdQQ/view?usp=sharing

### Installation

1. Install the plugin and enable it in **Edit → Plugins**.
2. Download and unzip the sample content.
3. Drop the unpacked folders into your project's `Content/` folder.
4. Done — open the sample map and play.

---

## FAQ

### Does the plugin work in Blueprint-only projects?

Yes. The DLLs are pre-compiled — no C++ project required. Just enable the plugin in **Edit → Plugins**.

### Do I need the sample content to use the plugin?

No. The sample is purely a starting point. The plugin works fine in any project — just add `UFPSWeaponFeelComponent` to your character under the camera.

---

## Author

**Lamedev** (Kyrylo Kabak)
[LinkedIn](https://www.linkedin.com/in/kyrylo-kabak-a3a2a5194)
Support: kyrylo.kabak@gmail.com
