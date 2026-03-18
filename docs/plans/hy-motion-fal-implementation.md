# Implementation Plan: HY-Motion via fal.ai API

**Date:** 2026-03-18
**Decision:** Use HY-Motion (Tencent Hunyuan) via [fal.ai hosted API](https://fal.ai/models/fal-ai/hunyuan-motion/api) instead of self-hosting
**Target:** Text prompt → 3D animation clip → Three.js (GLB, Mixamo skeleton)

---

## Why fal.ai + HY-Motion?

| Self-hosted HY-Motion | fal.ai HY-Motion |
|------------------------|------------------|
| Need A100/RTX 4090 (24-26GB VRAM) | No GPU infrastructure to manage |
| DevOps overhead (CUDA, model weights, scaling) | Pay-per-use, auto-scaling |
| ~$1-3/hr GPU cost even when idle | Pay only when generating |
| Full control over pipeline | Slightly less control, but simpler |
| Free per-generation | ~$0.05-0.15 per generation (estimated) |

**fal.ai provides:** Two model variants on their serverless GPU fleet:
- `fal-ai/hunyuan-motion` — 1B parameter model (highest quality)
- `fal-ai/hunyuan-motion/fast` — 0.46B parameter model (faster, lower VRAM)

---

## Architecture Overview

```
                    text prompt (e.g. "a person swinging a sword")
                            |
                            v
                  +--------------------+
                  |  Animation Router  |  Step 1: Check preset library first
                  +--------+-----------+
                           |
                    found? | not found?
                           |      |
                           v      v
                    return    +---------------------+
                    cached    | fal.ai API Client   |  Step 2: Call fal-ai/hunyuan-motion
                    GLB       | (subscribe pattern) |
                              +----------+----------+
                                         |
                                         v
                              +---------------------+
                              | SMPL-H Motion Data  |  Raw output: 22-joint SMPL-H skeleton
                              | (BVH / joint data)  |
                              +----------+----------+
                                         |
                                         v
                              +---------------------+
                              | Retarget Pipeline   |  Step 3: SMPL-H → Mixamo skeleton
                              | (server-side)       |  - Bone remapping (22 → 65 joints)
                              +----------+----------+  - T-pose alignment
                                         |            - Scale normalization
                                         v
                              +---------------------+
                              | GLB Export          |  Step 4: Export as GLB for Three.js
                              +----------+----------+
                                         |
                                         v
                              +---------------------+
                              | Cache + Serve       |  Step 5: Store in clip library, return URL
                              +---------------------+
```

---

## Implementation Steps

### Phase 0: Project Setup
> Estimated: foundation work

**0.1 — Initialize project structure**
```
src/
├── server/
│   ├── index.ts                    # Express/Fastify server entry
│   ├── config.ts                   # Environment config (FAL_KEY, etc.)
│   └── routes/
│       └── animations.ts           # POST /api/animations/generate
├── services/
│   ├── animation-router.ts         # Orchestrator: library check → generate → retarget → cache
│   ├── fal-client.ts               # fal.ai API wrapper
│   ├── retargeting/
│   │   ├── smpl-to-mixamo.ts       # SMPL-H → Mixamo bone mapping
│   │   └── glb-exporter.ts         # Export retargeted animation as GLB
│   └── clip-library.ts             # Preset library + cache management
├── client/
│   ├── animation-loader.ts         # Three.js GLTFLoader integration
│   ├── animation-mixer.ts          # Three.js AnimationMixer management
│   └── animation-state-machine.ts  # State machine for character animation
├── types/
│   └── animation.ts                # Shared type definitions
└── data/
    └── presets/                     # Pre-downloaded Mixamo GLB clips
        ├── idle.glb
        ├── walk.glb
        ├── run.glb
        └── ...
```

**0.2 — Install dependencies**
```json
{
  "dependencies": {
    "@fal-ai/client": "^1.9.4",
    "three": "^0.183.0",
    "@types/three": "^0.183.0",
    "express": "^4.21.0"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "@types/node": "^22.0.0"
  }
}
```

**0.3 — Environment configuration**
```env
FAL_KEY=your_fal_ai_api_key
ANIMATION_CACHE_DIR=./cache/animations
PRESET_LIBRARY_DIR=./src/data/presets
```

---

### Phase 1: fal.ai Client Integration
> Core: calling the HY-Motion model and handling the response

**1.1 — fal.ai API client wrapper** (`src/services/fal-client.ts`)

The fal.ai SDK uses a `subscribe` pattern with queue-based async processing:

```typescript
import { fal } from "@fal-ai/client";

// Configure authentication
fal.config({ credentials: process.env.FAL_KEY });

interface HYMotionInput {
  prompt: string;                    // Text description of motion (< 60 words, English)
  num_inference_steps?: number;      // Diffusion steps (default ~50, lower = faster)
  guidance_scale?: number;           // CFG scale (default ~7.5, higher = more prompt-adherent)
  seed?: number;                     // Reproducibility (-1 for random)
  duration?: number;                 // Motion duration in seconds (max ~10-12s)
}

interface HYMotionOutput {
  motion_file: {                     // BVH or NPZ motion file
    url: string;
    content_type: string;
    file_name: string;
    file_size: number;
  };
  video?: {                          // Optional preview video
    url: string;
    content_type: string;
  };
}

// Subscribe pattern: submits to queue, polls, returns result
const result = await fal.subscribe("fal-ai/hunyuan-motion", {
  input: {
    prompt: "a person swinging a sword with both hands",
    num_inference_steps: 50,
    guidance_scale: 7.5,
    seed: -1,
  },
  logs: true,
  onQueueUpdate: (update) => {
    if (update.status === "IN_PROGRESS") {
      console.log(update.logs?.map(l => l.message));
    }
  },
});
```

**Key considerations:**
- fal.ai queue pattern: `submit → IN_QUEUE → IN_PROGRESS → COMPLETED`
- Subscribe handles polling automatically
- Response likely contains a URL to a BVH/motion file hosted on fal.ai CDN
- Fast variant (`fal-ai/hunyuan-motion/fast`) uses the 0.46B model — use for previews/drafts

**1.2 — Input validation & prompt engineering**

HY-Motion works best with structured prompts:
- Keep under 60 words
- Use format: `"a person [action] [detail] [style]"`
- Good: `"a person walking forward slowly with arms swinging"`
- Bad: `"make the character do that cool thing from the movie"`

Create a prompt sanitizer that:
- Strips non-motion descriptors (camera angles, lighting, etc.)
- Ensures subject ("a person") is present
- Truncates to 60 words
- Optionally uses an LLM to refine vague prompts

**1.3 — Error handling & retries**

```typescript
// fal.ai specific error handling:
// - 429: Rate limited → exponential backoff (2s, 4s, 8s, 16s)
// - 503: Model cold start → retry after 10s (first request can be slow)
// - 504: Generation timeout → retry with fewer steps or shorter duration
// - Network errors → retry up to 4 times with backoff
```

---

### Phase 2: SMPL-H → Mixamo Retargeting Pipeline
> Critical path: converting HY-Motion's output skeleton to Mixamo format

**2.1 — Understanding the output format**

HY-Motion outputs **SMPL-H skeleton data** with 22 body joints:
```
Root (pelvis)
├── Left Hip → Left Knee → Left Ankle → Left Foot
├── Right Hip → Right Knee → Right Ankle → Right Foot
├── Spine 1 → Spine 2 → Spine 3
│   ├── Neck → Head
│   ├── Left Collar → Left Shoulder → Left Elbow → Left Wrist
│   └── Right Collar → Right Shoulder → Right Elbow → Right Wrist
```

Mixamo skeleton has **~65 joints** (including fingers, toes, twist bones):
```
mixamorig:Hips
├── mixamorig:LeftUpLeg → LeftLeg → LeftFoot → LeftToeBase
├── mixamorig:RightUpLeg → RightLeg → RightFoot → RightToeBase
├── mixamorig:Spine → Spine1 → Spine2
│   ├── mixamorig:Neck → Head → HeadTop_End
│   ├── mixamorig:LeftShoulder → LeftArm → LeftForeArm → LeftHand → [5 fingers × 3 joints]
│   └── mixamorig:RightShoulder → RightArm → RightForeArm → RightHand → [5 fingers × 3 joints]
```

**2.2 — Bone mapping table** (`src/services/retargeting/smpl-to-mixamo.ts`)

```typescript
const SMPL_TO_MIXAMO_MAP: Record<string, string> = {
  "Pelvis":           "mixamorig:Hips",
  "L_Hip":            "mixamorig:LeftUpLeg",
  "R_Hip":            "mixamorig:RightUpLeg",
  "Spine1":           "mixamorig:Spine",
  "L_Knee":           "mixamorig:LeftLeg",
  "R_Knee":           "mixamorig:RightLeg",
  "Spine2":           "mixamorig:Spine1",
  "L_Ankle":          "mixamorig:LeftFoot",
  "R_Ankle":          "mixamorig:RightFoot",
  "Spine3":           "mixamorig:Spine2",
  "L_Foot":           "mixamorig:LeftToeBase",
  "R_Foot":           "mixamorig:RightToeBase",
  "Neck":             "mixamorig:Neck",
  "L_Collar":         "mixamorig:LeftShoulder",
  "R_Collar":         "mixamorig:RightShoulder",
  "Head":             "mixamorig:Head",
  "L_Shoulder":       "mixamorig:LeftArm",
  "R_Shoulder":       "mixamorig:RightArm",
  "L_Elbow":          "mixamorig:LeftForeArm",
  "R_Elbow":          "mixamorig:RightForeArm",
  "L_Wrist":          "mixamorig:LeftHand",
  "R_Wrist":          "mixamorig:RightHand",
};
// Unmapped Mixamo joints (fingers, twist bones, toes) get identity/rest-pose transforms
```

**2.3 — Retargeting approach options**

| Approach | Pros | Cons | Recommendation |
|----------|------|------|----------------|
| **A: Server-side Blender CLI** | Most reliable, battle-tested, handles edge cases | Requires Blender installed on server (~200MB), Python subprocess | **Recommended for production** |
| **B: Server-side pure JS/TS** | No external deps, runs in Node.js | Must implement BVH parsing + retarget + GLB export from scratch | Good for MVP |
| **C: Client-side Three.js** | Zero server processing | SkeletonUtils.retargetClip has known bugs, adds client load time | Not recommended |

**Recommended: Approach A (Blender CLI) for production, Approach B for MVP**

**Approach B (MVP) implementation:**
1. Download the BVH/motion file from fal.ai CDN URL
2. Parse BVH (use `bvh-parser` npm package or custom parser)
3. Map SMPL-H joints → Mixamo joints using the bone table
4. For unmapped Mixamo joints (fingers etc.), use rest pose (identity quaternion)
5. Handle T-pose vs A-pose offset (SMPL-H uses T-pose, Mixamo uses slight A-pose — apply shoulder rotation offset)
6. Export as GLB using `@gltf-transform/core`

**Approach A (Production) implementation:**
1. Download BVH from fal.ai
2. Run headless Blender with a Python retargeting script:
   ```bash
   blender --background --python retarget_smpl_to_mixamo.py -- \
     --input motion.bvh \
     --output animation.glb \
     --target-rig mixamo
   ```
3. The Blender script imports BVH, uses Auto-Rig Pro or custom bone mapping, exports GLB
4. Return the GLB file

**2.4 — Post-processing**

After retargeting, apply:
- **Foot locking / IK cleanup**: Prevent foot sliding (common in generated motion)
- **Root motion extraction**: Separate root translation from animation for game-style movement
- **Loop blending**: If the clip needs to loop, blend first/last frames
- **Scale normalization**: Ensure output matches the game world scale

---

### Phase 3: Animation Clip Library & Caching
> Avoid regenerating common animations

**3.1 — Preset library** (`src/data/presets/`)

Pre-download and convert common Mixamo animations to GLB:

| Category | Clips | Source |
|----------|-------|--------|
| Locomotion | idle, walk, run, sprint, walk_back, strafe_left, strafe_right | Mixamo |
| Jumping | jump_start, jump_loop, jump_land | Mixamo |
| Combat | punch, kick, sword_swing, block, dodge | Mixamo |
| Social | wave, point, clap, sit_down, stand_up | Mixamo |
| Emotional | celebrate, cry, angry, scared | Mixamo |

Download from Mixamo as FBX → convert to GLB with Blender CLI batch script.

**3.2 — Semantic clip matching**

Before calling fal.ai, check if the prompt matches a preset:

```typescript
// Simple approach: keyword matching + cosine similarity
function findPresetMatch(prompt: string): PresetClip | null {
  // 1. Exact keyword match: "idle" → presets/idle.glb
  // 2. Synonym matching: "standing still" → idle, "jogging" → run
  // 3. Optional: Use a small embedding model for semantic similarity
  //    (e.g., fal-ai/text-embedding or local model)
}
```

**3.3 — Generation cache**

```typescript
// Cache key: normalized prompt hash + model variant + seed
// Storage: filesystem (./cache/animations/{hash}.glb)
// TTL: indefinite (animations don't expire)
// Eviction: LRU when cache exceeds size limit
```

---

### Phase 4: Three.js Client Integration
> Loading and playing generated animations

**4.1 — Animation loader** (`src/client/animation-loader.ts`)

```typescript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { AnimationClip, AnimationMixer } from "three";

async function loadAnimation(url: string): Promise<AnimationClip> {
  const loader = new GLTFLoader();
  const gltf = await loader.loadAsync(url);
  // GLB from our pipeline will contain exactly one AnimationClip
  return gltf.animations[0];
}
```

**4.2 — Animation state machine** (`src/client/animation-state-machine.ts`)

```typescript
// States map to animation clips
type AnimState = "idle" | "walk" | "run" | "jump" | "custom";

interface Transition {
  from: AnimState;
  to: AnimState;
  duration: number;        // Crossfade duration in seconds
  condition: () => boolean;
}

// The state machine:
// 1. Manages active AnimationAction on the Three.js AnimationMixer
// 2. Handles crossfade transitions between states
// 3. Supports "custom" state for AI-generated clips
// 4. Falls back to "idle" if a clip fails to load
```

**4.3 — Async generation flow (client-side UX)**

```
User types prompt → "Generate" button
  ↓
POST /api/animations/generate { prompt }
  ↓
Client shows loading indicator (skeleton preview animation)
  ↓
Server: check cache → check presets → call fal.ai → retarget → export GLB → cache
  ↓
Response: { clipUrl: "/animations/abc123.glb", duration: 3.2, cached: false }
  ↓
Client: GLTFLoader.load(clipUrl) → play on character
```

---

### Phase 5: Server API
> Express/Fastify endpoint orchestrating the pipeline

**5.1 — API endpoint** (`src/server/routes/animations.ts`)

```
POST /api/animations/generate
  Body: {
    prompt: string,           // Required: motion description
    model?: "1b" | "fast",    // Default: "1b". Use "fast" for previews
    seed?: number,            // Default: -1 (random)
    duration?: number,        // Default: auto. Max 10s
    steps?: number,           // Default: 50. Lower = faster but lower quality
    guidance?: number,        // Default: 7.5
    useCache?: boolean,       // Default: true
  }
  Response: {
    clipUrl: string,          // URL to serve the GLB file
    duration: number,         // Animation duration in seconds
    cached: boolean,          // Whether this was served from cache
    generationTime?: number,  // ms spent generating (if not cached)
    model: string,            // Which model variant was used
  }
```

**5.2 — Orchestration flow** (`src/services/animation-router.ts`)

```typescript
async function generateAnimation(request: GenerateRequest): Promise<GenerateResponse> {
  // Step 1: Check cache
  const cached = await clipLibrary.findCached(request.prompt, request.seed);
  if (cached && request.useCache !== false) return cached;

  // Step 2: Check preset library
  const preset = await clipLibrary.findPreset(request.prompt);
  if (preset) return preset;

  // Step 3: Call fal.ai HY-Motion
  const endpoint = request.model === "fast"
    ? "fal-ai/hunyuan-motion/fast"
    : "fal-ai/hunyuan-motion";

  const falResult = await falClient.generate(endpoint, {
    prompt: sanitizePrompt(request.prompt),
    num_inference_steps: request.steps ?? 50,
    guidance_scale: request.guidance ?? 7.5,
    seed: request.seed ?? -1,
    duration: request.duration,
  });

  // Step 4: Download motion file from fal.ai CDN
  const motionData = await downloadMotionFile(falResult.motion_file.url);

  // Step 5: Retarget SMPL-H → Mixamo
  const mixamoAnimation = await retargetToMixamo(motionData);

  // Step 6: Export as GLB
  const glbBuffer = await exportGLB(mixamoAnimation);

  // Step 7: Cache and serve
  const clipUrl = await clipLibrary.store(glbBuffer, request.prompt, request.seed);

  return { clipUrl, duration: mixamoAnimation.duration, cached: false };
}
```

---

## Phase Summary & Milestones

| Phase | Description | Deliverable | Dependencies |
|-------|-------------|-------------|--------------|
| **0** | Project setup, deps, env config | Bootable project with types | None |
| **1** | fal.ai client integration | Can call HY-Motion, get raw motion data | FAL_KEY |
| **2** | SMPL-H → Mixamo retarget pipeline | BVH/motion → Mixamo-rigged GLB | Phase 1 output format confirmed |
| **3** | Preset library + caching | Curated Mixamo clips served as GLB, cache layer | Blender for conversion |
| **4** | Three.js client (loader + state machine) | Character plays animation clips in browser | Phase 2 GLBs |
| **5** | Server API endpoint | REST endpoint orchestrating full pipeline | Phases 1-3 |

**Suggested build order:** 0 → 1 → 2 → 3 → 5 → 4

Phase 1 should be done first because the **exact fal.ai output format** (BVH? NPZ? joint arrays? URLs vs inline data?) determines how Phase 2 is implemented. Run a test generation first, inspect the output, then build the retargeting pipeline accordingly.

---

## Open Questions (to resolve in Phase 1)

1. **What exactly does `fal-ai/hunyuan-motion` return?**
   - File URL to BVH? NPZ? Raw joint arrays in JSON?
   - The fal.ai API page (blocked from this env) has the schema — test with a real API call first
   - Run: `fal.subscribe("fal-ai/hunyuan-motion", { input: { prompt: "a person walking" } })` and inspect

2. **Does fal.ai do any post-processing?**
   - Some fal.ai model wrappers add conversion steps (e.g., outputting GLB directly)
   - If they already output BVH, Phase 2 is simpler
   - If they output raw SMPL-H NPZ, we need the full conversion pipeline

3. **Pricing confirmation**
   - fal.ai charges per-second of GPU time — need to benchmark cost per generation
   - The 1B model likely costs more than the 0.46B fast variant
   - Estimate: $0.05-0.20 per generation (based on similar fal.ai model costs)

4. **Latency benchmarks**
   - Cold start (model loading): potentially 30-60s for first request
   - Warm inference: likely 5-15s per generation
   - Use `keep_alive` or schedule warm-up requests to avoid cold starts

5. **T-pose vs A-pose alignment**
   - SMPL-H uses a specific rest pose — need to determine exact offset angles for Mixamo mapping
   - Test with a simple "T-pose" prompt and compare joint orientations

---

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| fal.ai output format is raw NPZ (hardest to convert) | High — need full SMPL conversion pipeline | Prototype Phase 1 first; if NPZ, add `smpl2bvh` step |
| Retargeting quality issues (foot sliding, jitter) | Medium — affects animation quality | Implement foot IK post-processing; allow quality review before caching |
| fal.ai cold start latency (~30-60s) | Medium — bad first-request UX | Pre-warm the model with a scheduled ping; use fast variant for previews |
| fal.ai rate limits or downtime | Medium — blocks generation | Queue with retry; fallback to preset library; consider SayMotion as backup provider |
| HY-Motion license (no EU/UK/South Korea) | High — legal risk | Verify with fal.ai whether their hosting changes license terms; consult legal |
| Finger/hand animation missing (SMPL-H has 22 joints, no fingers) | Low — acceptable for most game animations | Use rest-pose hands; optionally blend with hand-specific clips |

---

## Next Step

**Run a test generation against the fal.ai API** to confirm the output format:

```typescript
import { fal } from "@fal-ai/client";

fal.config({ credentials: "YOUR_FAL_KEY" });

const result = await fal.subscribe("fal-ai/hunyuan-motion", {
  input: { prompt: "a person walking forward" },
  logs: true,
  onQueueUpdate: (u) => console.log(u.status, u.logs),
});

console.log(JSON.stringify(result, null, 2));
// ^^^ Inspect this output to determine exact schema before Phase 2
```

This single API call will answer Open Questions 1-2 and unblock the retargeting pipeline design.
