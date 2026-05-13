# Changelog

All notable changes to **FPS Weapon Feel** are documented here.
This project adheres to [Semantic Versioning](https://semver.org/) and the format of [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- `UFPSWeaponFeelComponent::InitializeRecoilData(const FRecoilProfile&)` — caches the per-weapon recoil profile. Call once on weapon equip / profile change instead of passing the profile on every shot.

### Changed
- **Breaking:** `FireShot` no longer takes an `FRecoilProfile` argument — it reads the profile cached by `InitializeRecoilData`. Migration: call `InitializeRecoilData(Profile)` on equip, then `FireShot()` per bullet.

## [1.0.0] — 2026-05-11

Initial release.

### Added
- `UFPSWeaponFeelComponent` — single component encapsulating procedural FPS weapon motion.
- Rotation lag with framerate-independent angular-velocity normalization.
- Movement tilt and lag driven by character velocity in local space.
- Idle breathing sway with movement-aware amplitude.
- Walk bob locked to character ground state and speed.
- Per-shot recoil with view kick (direct `SetControlRotation`, sensitivity-invariant) and mesh kick (offset + tilt).
- `FRecoilProfile` struct carrying per-weapon recoil tuning.
- `URecoilPattern` asset and custom 2D editor for authoring weapon spray curves.
- True exponential-smoothing interpolation across all recovery channels — identical motion at any frame rate.
- `Intensity` and `IntensityInterpSpeed` for smooth ADS fade.
- `ResetState()` to clear all accumulated motion on respawn / teleport.
- `bEnabled` toggle that keeps tracking parent rotation while disabled, so re-enabling never produces a one-frame spike.
- Pattern fallback after the last authored point — gracefully degrades to `ViewPitchKick + ViewYawSpread` instead of repeating the final delta.
- Network safety — Tick and FireShot are no-ops on remote proxies and the server.

### Targets
- Unreal Engine 5.7
- Win64
