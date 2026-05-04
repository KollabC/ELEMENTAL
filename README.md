# ElementalClash — Architecture Document

## 1. Overview

**ElementalClash** is a 2-player local PvP browser game built as a single HTML file using vanilla JS + Three.js + Rapier.js + Howler.js. Designed for portrait mobile (iPhone 13 Pro Max — 390×844px). All geometry is procedural; no external assets or build step.

The game is a turn-based spell-casting duel where two players select elements from a radial spell wheel, launch projectiles at each other, and resolve counters using rock-paper-scissors logic.

---

## 2. Module Map

| Module | Responsibility | Public API | Dependencies |
|---|---|---|---|
| `GameStateMachine` | State transitions, game flow control | `enter(state)`, `update(dt)` | all systems |
| `SceneSetup` | Three.js scene, renderer, camera, resize | `init()`, `resize()`, `render()` | Three.js |
| `ArenaGeometry` | Procedural floor, rings, cubes, vortex, camps, torches | `build()`, `animate(time)` | Three.js |
| `LightingRig` | Ambient, point lights, torch flicker | `init()`, `update(time)` | Three.js |
| `SpellWheelUI` | HTML overlay wheel, touch handling | `init()`, `selectSpell(player, element)`, `showCooldown(slot, pct)` | DOM |
| `SpellManager` | Element configs, cooldown, spell data | `cast(player, element)`, `getCooldown(slot)` | — |
| `SpellConfig` | Element data constants | `ELEMENTS`, `COUNTERS`, `DAMAGE_TABLE` | — |
| `ProjectileManager` | Spawn, update, destroy projectiles | `spawn()`, `update(dt)`, `checkCollisions()` | Three.js, Rapier |
| `Projectile` | Per-element projectile entity | `launch()`, `update()`, `onHit()`, `onCounter()` | Three.js |
| `ParticleSystem` | InstancedMesh particles per element | `emit(pos, count)`, `update(dt)` | Three.js |
| `HPUIManager` | HP bars, damage numbers, health logic | `init()`, `setHP(player, hp)`, `showDamage(player, amt)` | DOM |
| `AudioManager` | Howler.js music + SFX | `init()`, `playMusic(track)`, `playSFX(sfx)` | Howler.js |
| `CinematicEditor` | Timeline, keyframes, camera interpolation | `addKeyframe(state)`, `play()`, `exportJSON()` | Three.js |
| `CinematicTimeline` | DOM timeline UI | `render()`, `onScrub(pct)` | DOM |
| `PostProcessing` | Bloom, screen shake | `init()`, `update()`, `shake(intensity)` | Three.js |
| `LoadingScreen` | Asset loading progress | `init()`, `setProgress(pct)` | DOM |

---

## 3. Class Tree

```
GameStateMachine
├── MENU
├── ROUND_INTRO
├── COMBAT
│   ├── PlayerState (P1, P2)
│   └── TurnManager
├── COUNTER_RESOLUTION
├── VICTORY
└── REPLAY

SceneSetup
├── WebGLRenderer
├── OrthographicCamera (isometric)
├── Scene
└── ResizeHandler

ArenaGeometry
├── CircularFloor
├── RunicRing × 3
├── DataCube × N
├── CentralVortex
├── PlayerCamp × 2
└── ArcaneTorch × 6

LightingRig
├── AmbientLight
├── PointLight × 2 (camps)
└── TorchFlickerSystem

SpellWheelUI
├── WheelHTML (P1 / P2)
│   ├── SpellSlot × 5
│   ├── CooldownRing × 5
│   └── CASTButton
└── TouchHandler

SpellManager
├── SpellConfig
│   ├── FIRE    { dmg:100, cooldown:2.0, radius:0.3, speed:8 }
│   ├── WATER   { dmg:80,  cooldown:1.5, radius:0.4, speed:6 }
│   ├── EARTH   { dmg:120, cooldown:3.0, radius:0.5, speed:4 }
│   ├── AIR     { dmg:60,  cooldown:1.0, radius:0.25, speed:12 }
│   └── LIGHTNING { dmg:150, cooldown:4.0, radius:0.35, speed:10 }
└── CooldownTracker

ProjectileManager
├── Projectile[0..MAX]
│   ├── FIRE     { color:0xff4400, mesh: sphere, particleTrail }
│   ├── WATER    { color:0x0088ff, mesh: sphere, splash }
│   ├── EARTH    { color:0x886622, mesh: box, debris }
│   ├── AIR      { color:0xaaddff, mesh: torus, swirl }
│   └── LIGHTNING { color:0xffff00, mesh: cylinder, sparks }
└── CollisionResolver

ParticleSystem
├── FireParticleSystem   (InstancedMesh, 200 instances)
├── WaterParticleSystem  (InstancedMesh, 200 instances)
├── EarthParticleSystem  (InstancedMesh, 200 instances)
├── AirParticleSystem    (InstancedMesh, 200 instances)
└── LightningParticleSystem (InstancedMesh, 200 instances)

HPUIManager
├── HPBar × 2
└── DamagePopup[]

AudioManager
├── Howl music
└── Howl sfx (sprite)

CinematicEditor
├── KeyframeManager
│   └── Keyframe[]
│       ├── time: number
│       ├── state: string
│       ├── cameraPosition: Vector3
│       └── cameraTarget: Vector3
├── TransitionEngine (cut / ease-in / ease-out / lerp)
└── TimelinePlayback

PostProcessing
├── EffectComposer
├── UnrealBloomPass
└── ScreenShake

GameLoop
├── update(dt)
└── render()
```

---

## 4. Data Structures

### Element Enum
```js
const ELEMENT = { FIRE: 0, WATER: 1, EARTH: 2, AIR: 3, LIGHTNING: 4 };
```

### Counter Matrix
```js
// 1.5× bonus vs weak, 0.5× penalty vs strong, 1× neutral
const COUNTER_MATRIX = [
  //FIRE  WATER  EARTH  AIR   LIGHT
  [1.0,  0.5,   1.5,   1.0,  0.5],  // FIRE
  [1.5,  1.0,   0.5,   1.0,  1.0],  // WATER
  [0.5,  1.5,   1.0,   1.0,  1.0],  // EARTH
  [1.0,  1.0,   1.0,   1.0,  1.5],  // AIR
  [1.5,  1.0,   1.0,   0.5,  1.0],  // LIGHTNING
];
```

### Game States
```js
const STATE = {
  MENU: 'menu',
  ROUND_INTRO: 'round_intro',
  COMBAT: 'combat',
  COUNTER_RESOLUTION: 'counter_res',
  VICTORY: 'victory',
  REPLAY: 'replay'
};
```

### Spell Config
```js
{
  id: number,
  name: string,
  damage: number,
  cooldownTime: number,
  radius: number,
  speed: number,
  color: hex,
  particleType: string
}
```

### Projectile
```js
{
  id: string,
  owner: 1 | 2,
  element: ELEMENT,
  position: Vector3,
  velocity: Vector3,
  damage: number,
  active: boolean,
  mesh: THREE.Mesh,
  createdAt: number
}
```

### Keyframe
```js
{
  id: string,
  time: number,        // seconds in timeline
  state: string,       // game state that triggers
  cameraPosition: { x, y, z },
  cameraRotation: { x, y, z },
  transitionType: 'cut' | 'ease-in' | 'ease-out' | 'lerp',
  duration: number
}
```

---

## 5. Game Flow

```
MENU → [Start Button] → ROUND_INTRO
ROUND_INTRO → [3s cinematic] → COMBAT
COMBAT → [Both players cast] → COUNTER_RESOLUTION
COUNTER_RESOLUTION → [Animate results] → Check HP
  ├─ HP > 0 → COMBAT (next round)
  └─ HP ≤ 0 → VICTORY
VICTORY → [Play again] → REPLAY
REPLAY → [Rematch] → ROUND_INTRO
```

---

## 6. Technical Spec

- **Target FPS**: 60
- **Max triangles**: 50,000
- **Particle instances**: 200 per element = 1,000 total (InstancedMesh)
- **Projectile pool**: max 20 active
- **Camera**: isometric top-down, 55° tilt, orthographic-ish perspective
- **Arena**: circular, radius 8 units
- **Spell wheel**: HTML overlay, touch events only
- **Audio**: Howler.js with sprite sheet
- **Physics**: Rapier.js rigid bodies for projectile collision
- **Post-processing**: Three.js UnrealBloomPass + manual screen shake

---

## 7. CDN Dependencies

```
Three.js:              https://cdnjs.cloudflare.com/ajax/libs/three.js/r134/three.min.js
Three.js EffectComposer: https://cdn.jsdelivr.net/npm/three@0.134.0/examples/js/postprocessing/EffectComposer.js
Three.js RenderPass:   https://cdn.jsdelivr.net/npm/three@0.134.0/examples/js/postprocessing/RenderPass.js
Three.js UnrealBloom:  https://cdn.jsdelivr.net/npm/three@0.134.0/examples/js/postprocessing/UnrealBloomPass.js
Three.js ShaderPass:   https://cdn.jsdelivr.net/npm/three@0.134.0/examples/js/postprocessing/ShaderPass.js
Howler.js:             https://cdnjs.cloudflare.com/ajax/libs/howler/2.2.3/howler.min.js
@dimforge/rapier3d:    https://cdn.skypack.dev/@dimforge/rapier3d@0.7.0
```
