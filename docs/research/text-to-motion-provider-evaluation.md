# Text-to-Motion Provider Evaluation

**Date:** 2026-03-18
**Context:** Roam AI game generation platform — need text prompt -> animation clips for humanoid characters, Three.js r183, Mixamo skeleton

---

## Executive Summary

**Recommended approach: Tiered strategy**

| Tier | Provider | Use Case |
|------|----------|----------|
| **Primary (preset library)** | Mixamo + curated clip library | Common clips (idle, walk, run, jump) — instant, zero cost, perfect Mixamo compat |
| **Primary (generation)** | **DeepMotion SayMotion API** | Text-to-motion for custom/uncommon clips — production-ready REST API, GLB/FBX/BVH output |
| **Fallback / future** | **HY-Motion 1.0 (self-hosted)** | Best open-source quality, self-hosted for cost control at scale |
| **Experimentation** | Mootion API | Broader creative motions, video generation pipeline |

**Why SayMotion wins for this POC:**
1. Only provider with a **documented REST API** (GitHub repo with examples) that directly takes text -> GLB
2. Outputs in **FBX, GLB, BVH** — GLB is what Three.js GLTFLoader consumes directly
3. Uses standard humanoid rig compatible with Mixamo retargeting
4. Supports custom character upload (FBX, GLB, VRM) with auto-retargeting
5. 400K+ clip dataset, 10s max per clip, multiple variants per generation
6. Async job model fits our pipeline (submit -> poll -> download)

---

## Detailed Provider Evaluations

### TIER 1: Production-Ready APIs

---

### 1. DeepMotion SayMotion (RECOMMENDED)

| Attribute | Detail |
|-----------|--------|
| **Type** | Hosted SaaS, REST API |
| **API** | Yes — [GitHub: SayMotion/REST-API](https://github.com/SayMotion/REST-API) |
| **Input** | Text prompt (subject + action + detail), max 10s per clip |
| **Output** | FBX, GLB, BVH, MP4 |
| **Skeleton** | Standard humanoid rig, Mixamo-compatible retargeting built in |
| **Latency** | Async job model — submit, poll, download. Seconds to minutes depending on queue |
| **Quality** | Good — 400K+ clip dataset (v1.5+), simulation for physics, foot locking options |
| **Pricing** | Credit-based: 1 credit = 10s animation. Free: 25 credits/mo (non-commercial). Paid: $15/mo (50 credits) to $300/mo (1000 credits). API: contact for enterprise |
| **Status** | GA (v2.4+), API in partner access with free trials available |
| **Strengths** | Mature API, GLB native output, inpainting/merging/looping features, custom character retarget |
| **Weaknesses** | API access requires partner verification, credits can get expensive at scale, 10s max clip length |

**API Flow:**
```
1. POST auth -> get session token
2. POST text2motion job { prompt, variants: 1-8, model_id? }
3. GET job status (poll)
4. GET download URLs -> .glb, .fbx, .bvh files
```

**Key features for our use case:**
- Multiple variants per generation (1-8) — pick best quality
- Skip FBX generation (`skipFBX=1`) to speed up if only GLB needed
- Loop mode with INTERPOLATION or LOCKED blending
- Inpainting: extend/modify clips with additional text prompts
- Custom character upload for retargeting

---

### 2. Kinetix

| Attribute | Detail |
|-----------|--------|
| **Type** | Hosted SaaS, SDK + API |
| **API** | Yes — REST API + Unity/Unreal SDKs. [docs.kinetix.tech](https://docs.kinetix.tech/integration/kinetix-api) |
| **Input** | Primarily video-to-animation; text-to-emotes (Text2Emotes) launching 2025-2026 |
| **Output** | FBX, glTF, proprietary Kinanim format |
| **Skeleton** | Mesh-aware retargeting, adapts to diverse avatar systems |
| **Pricing** | $0.20 per emote generated |
| **Status** | GA for video-to-animation; Text2Emotes in beta/upcoming |
| **Strengths** | Adobe Mixamo partnership, Unity Muse integration, 1000+ emote library, game-focused |
| **Weaknesses** | Text-to-motion not fully available via API yet, primarily video-input focused, per-generation pricing adds up |

**Verdict:** Strong for UGC emotes in games, but text-to-motion API not production-ready yet. Watch for Text2Emotes API release. Good as a supplementary source for their emote library.

---

### 3. Mootion

| Attribute | Detail |
|-----------|--------|
| **Type** | Hosted SaaS, API available |
| **API** | Yes — REST API at [mootion.com/api.html](https://www.mootion.com/api.html) |
| **Input** | Text, video, or audio |
| **Output** | FBX, GLB |
| **Skeleton** | Rig-free motion system |
| **Pricing** | Free: 200 credits/mo. Standard: ~$12/mo (1000 credits). Pro: ~$48/mo (5000 credits) |
| **Status** | GA (v4.0 — evolved into broader video generation platform) |
| **Strengths** | Generous free tier, multi-modal input, good export formats |
| **Weaknesses** | Pivoted heavily toward video generation (Mootion 4.0), less focused on pure 3D motion clips. API docs not publicly detailed |

**Verdict:** Interesting for broader creative use but has diverged from pure text-to-3D-motion. API exists but documentation is sparse. Worth testing for motion quality.

---

### 4. Rokoko

| Attribute | Detail |
|-----------|--------|
| **Type** | Desktop app (Rokoko Studio) + hardware mocap |
| **API** | No public text-to-motion API. Feature is in Rokoko Studio Preview desktop app only |
| **Input** | Text prompts (in Studio app), video (Vision), hardware mocap |
| **Output** | FBX, BVH, C3D + streaming to Blender/Unity/Unreal |
| **Quality** | High — trained on millions of unique motion assets, produces clean base motions |
| **Pricing** | Paid plans required for text-to-motion. Free trial available |
| **Status** | Text-to-motion in Studio Preview (2024-2025) |
| **Strengths** | Professional-grade quality, designed for production pipelines, excellent integrations |
| **Weaknesses** | No REST API for text-to-motion, locked to desktop app |

**Verdict:** Excellent quality but no programmatic API. Not viable for our automated pipeline.

---

### 5. Plask

| Attribute | Detail |
|-----------|--------|
| **Type** | Web-based SaaS, SDK available |
| **API** | SDK for integration; text-to-motion capability exists but API details unclear |
| **Input** | Text prompts, video |
| **Output** | FBX, GLB, BVH |
| **Quality** | 90-95% motion detail from video; text-to-motion quality unclear |
| **Pricing** | Free tier for testing, paid plans for production |
| **Status** | GA |
| **Strengths** | Claims "1st text-to-motion AI", browser-based, good export formats |
| **Weaknesses** | More focused on outsourcing/studio workflow than developer API, limited developer docs |

**Verdict:** Interesting but API/SDK documentation insufficient for evaluation. Not recommended for automated pipeline without further investigation.

---

### 6. Cartwheel (WATCH LIST)

| Attribute | Detail |
|-----------|--------|
| **Type** | Hosted SaaS, browser-based |
| **API** | Planned but NOT yet publicly available |
| **Team** | Pixar/Riot Games veterans, backed by Jeffrey Katzenberg |
| **Funding** | $15.6M |
| **Input** | Text prompts (e.g. "cast a wizard spell", "do a silly dance") |
| **Output** | Export to Maya, Unity, Unreal Engine (formats unclear) |
| **Quality** | Used by DreamWorks, Duolingo, Sony, Roblox during beta. Powered by Google Gemini |
| **Status** | Emerged from beta May 2025, 60,000+ waitlist |

**Verdict:** Very promising team and backers, but no API yet. Worth monitoring — could become the best option once API ships.

---

### 7. Meshy (3D generation, NOT text-to-motion)

| Attribute | Detail |
|-----------|--------|
| **Type** | Hosted SaaS, REST API |
| **API** | Yes — [docs.meshy.ai](https://docs.meshy.ai). Text-to-3D, Image-to-3D, Auto-rigging + Animation |
| **Input** | Text prompt for 3D model generation. Animation from 500+ pre-made library |
| **Output** | GLB, FBX, OBJ, USDZ |
| **Pricing** | Free: 100 credits/mo. Pro: $16/mo (1000 credits). Max: $48/mo (4000 credits) |
| **Status** | GA (Meshy-6) |

**Verdict:** NOT a text-to-motion generator — it generates 3D models from text, then applies pre-made animations from a library. Useful for generating character models but not for custom motion generation.

---

### TIER 2: Self-Hosted Open Source

---

### 6. HY-Motion 1.0 (Tencent Hunyuan) — BEST OPEN SOURCE

| Attribute | Detail |
|-----------|--------|
| **Type** | Self-hosted, open-source |
| **Code** | [GitHub: Tencent-Hunyuan/HY-Motion-1.0](https://github.com/Tencent-Hunyuan/HY-Motion-1.0) |
| **Models** | 1.0B params (standard) / 0.46B params (lite) — [HuggingFace](https://huggingface.co/tencent/HY-Motion-1.0) |
| **Input** | Text prompt |
| **Output** | SMPL-H 22-joint skeleton -> FBX, BVH, GLB via export tools |
| **GPU** | 26 GB VRAM (standard) / 24 GB VRAM (lite) — A100 or RTX 4090 |
| **Quality** | State-of-the-art — billion-parameter model, 200+ motion categories, 3000+ hours training data |
| **Skeleton** | SMPL-H 22 joints, requires retargeting to Mixamo |
| **Retarget** | Community tools: [hy-motion-fbx-exporter](https://github.com/zysilm-ai/hy-motion-fbx-exporter) for Mixamo retarget |
| **License** | Tencent community license — NOT available in EU, UK, South Korea |
| **Max Duration** | Clips < 12 seconds, 30 fps |

**Strengths:**
- Best quality among open-source options by a significant margin
- 200+ motion categories across 6 major classes
- Reinforcement learning from human feedback (RLHF) improves output quality
- ComfyUI plugins available for easy workflow integration
- Can be wrapped in a FastAPI service

**Weaknesses:**
- Requires beefy GPU (24-26 GB VRAM)
- SMPL-H -> Mixamo retargeting adds pipeline complexity
- Geographic license restrictions (no EU/UK/South Korea)
- Self-hosting operational overhead

**Verdict:** Best quality open-source option. Ideal for scale (no per-generation cost) if you can host GPU infrastructure. License restrictions may be a concern. **Recommended as secondary/future provider** once POC validates with SayMotion.

---

### 7. MoMask

| Attribute | Detail |
|-----------|--------|
| **Type** | Self-hosted, open-source (research) |
| **Code** | [GitHub: EricGuo5513/momask-codes](https://github.com/EricGuo5513/momask-codes) (~1.1k stars) |
| **Quality** | FID 0.045 on HumanML3D (excellent for research benchmark) |
| **Output** | .npy motion arrays (requires conversion pipeline) |
| **Skeleton** | HumanML3D format, requires retargeting |
| **Weaknesses** | Limited diversity, struggles with fast root motion, research-grade output format |

**Verdict:** Good research model but output requires significant pipeline work to get to usable GLB. HY-Motion 1.0 supersedes it.

---

### 8. MDM (Motion Diffusion Model)

| Attribute | Detail |
|-----------|--------|
| **Type** | Self-hosted, open-source (research) |
| **Code** | [GitHub: GuyTevet/motion-diffusion-model](https://github.com/GuyTevet/motion-diffusion-model) |
| **Quality** | Pioneering but surpassed by newer models |
| **Weaknesses** | Many diffusion steps = slow inference, some unrealistic motions, .npy output |

**Verdict:** Historical significance but superseded by MoMask and HY-Motion.

---

### 9. MotionGPT / MotionGPT3

| Attribute | Detail |
|-----------|--------|
| **Type** | Self-hosted, open-source |
| **Code** | [GitHub: OpenMotionLab/MotionGPT](https://github.com/OpenMotionLab/MotionGPT) (~1.4k stars) |
| **MotionGPT3** | [GitHub: OpenMotionLab/MotionGPT3](https://github.com/OpenMotionLab/MotionGPT3) — MIT License, 2025 |
| **Quality** | State-of-the-art for multi-task (generation + captioning + prediction) |
| **Output** | .npy motion arrays |
| **Weaknesses** | Complex setup (SMPL, GloVe, CLIP, spaCy), .npy output needs conversion |

**Verdict:** Interesting multi-modal capabilities but pipeline complexity is high. HY-Motion is better for pure generation quality.

---

## Format Conversion Pipeline

### FBX/BVH -> GLB (for Three.js)

| Tool | Method | Notes |
|------|--------|-------|
| **Blender CLI** | `blender --background --python convert.py` | Most reliable, handles animations well. Can script import FBX -> export GLB |
| **FBX2GLTF** | CLI tool by Facebook | Fast, but can lose some animation data |
| **pygltflib** | Python library | Good for programmatic GLB manipulation, less good for FBX import |
| **Three.js FBXLoader -> GLTFExporter** | In-browser | Works but adds client-side complexity |

**Recommended pipeline:**
```
SayMotion API -> GLB (native) -> serve directly
  OR
HY-Motion -> SMPL-H .npy -> hy-motion-fbx-exporter -> FBX (Mixamo rigged) -> Blender CLI -> GLB
```

### Skeleton Retargeting to Mixamo

| Approach | Tool | Notes |
|----------|------|-------|
| **SayMotion built-in** | Upload custom char, auto-retarget | Best for SayMotion pipeline |
| **HY-Motion -> Mixamo** | hy-motion-fbx-exporter | CLI tool, handles SMPL-H -> Mixamo mapping |
| **Three.js runtime** | `SkeletonUtils.retargetClip()` | Client-side retarget, bone naming must match |
| **Blender script** | Rokoko or Mixamo retarget addons | Offline batch processing |

---

## Preset Library Strategy (Mixamo + Community)

For common clips, avoid generation entirely:

| Source | Clips Available | Format | License | Notes |
|--------|----------------|--------|---------|-------|
| **Mixamo** | 2500+ animations | FBX | Free for commercial use | No programmatic API — need batch download tool or pre-download |
| **Quaternius Universal Animation Library** | 100+ common animations | GLB | CC0 | Free, already in GLB, Mixamo-compatible rig |
| **Kinetix Emote Library** | 1000+ emotes | glTF | Commercial license | Game-focused emotes |

**Recommended:** Pre-download a curated Mixamo clip set for each archetype (idle, walk, run, sprint, jump, land, etc.), convert to GLB, and serve from library. Only hit the generation API for uncommon/custom clips.

---

## Benchmark Comparison

| Provider | Quality (1-10) | API Readiness | Latency | Cost per Clip | Mixamo Compat | GLB Native |
|----------|----------------|---------------|---------|---------------|---------------|------------|
| **SayMotion** | 7 | 9/10 (REST API, documented) | ~15-25s | ~$0.30-$3.00 | Good (built-in retarget) | Yes |
| **HY-Motion 1.0** | 9 | 5/10 (self-host, wrap in API) | ~10-30s (on GPU) | GPU cost only | Needs retarget tool | Via export tool |
| **Kinetix** | 7 | 6/10 (text API upcoming) | Unknown | $0.20/emote | Excellent (Mixamo partner) | glTF yes |
| **Mootion** | 7 | 5/10 (API exists, sparse docs) | Unknown | ~$0.01-0.05/credit | Good | Yes |
| **Rokoko** | 8 | 1/10 (no API) | N/A | Subscription | Good | Via export |
| **MoMask** | 6 | 3/10 (research code) | ~5-15s | GPU cost only | Needs pipeline | No |
| **MDM** | 5 | 3/10 (research code) | ~30-60s | GPU cost only | Needs pipeline | No |

---

## Recommended Architecture for Roam POC

```
                  text prompt
                      |
                      v
            +-------------------+
            |  Clip Library     |  <-- Check first: is this a common clip?
            |  (Mixamo presets  |      idle, walk, run, jump, etc.
            |   + cached gens)  |
            +--------+----------+
                     |
              found? |  not found?
                     |      |
                     v      v
              return     +------------------+
              cached     | SayMotion API    |  <-- Text-to-motion generation
              GLB        | POST text2motion |
                         +--------+---------+
                                  |
                                  v
                         +------------------+
                         | GLB clip (native)|  <-- Already in Three.js format
                         +--------+---------+
                                  |
                                  v
                         +------------------+
                         | Validate bones   |  <-- Check Mixamo hierarchy
                         | Cache in library |
                         +------------------+
```

**Phase 2 (scale):** Add HY-Motion 1.0 self-hosted as generation backend for high-volume/cost optimization. Requires GPU infrastructure (A100/RTX 4090).

---

## Key Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| SayMotion API access requires partner verification | Apply early, use web portal as fallback for POC |
| Generation latency (30-120s) blocks game creation | Pre-generate common clips, async generation, show progress UI |
| Credit costs at scale | Aggressive caching, preset library for common clips, HY-Motion self-hosted for Phase 2 |
| Animation quality inconsistency | Generate multiple variants (up to 8), pick best, allow re-generation in previewer |
| Mixamo retarget failures on edge cases | Validate bone hierarchy server-side, fallback to closest preset clip |
| HY-Motion license restrictions (no EU/UK) | Use SayMotion for EU/UK users, or evaluate license implications with legal |

---

## Sources

- [SayMotion REST API (GitHub)](https://github.com/SayMotion/REST-API)
- [DeepMotion SayMotion](https://www.deepmotion.com/saymotion)
- [SayMotion API page](https://www.deepmotion.com/saymotion-api)
- [SayMotion Pricing](https://www.deepmotion.com/pricing-saymotion)
- [HY-Motion 1.0 (GitHub)](https://github.com/Tencent-Hunyuan/HY-Motion-1.0)
- [HY-Motion 1.0 (HuggingFace)](https://huggingface.co/tencent/HY-Motion-1.0)
- [hy-motion-fbx-exporter](https://github.com/zysilm-ai/hy-motion-fbx-exporter)
- [HY-Motion GPU Requirements Deep Dive](https://blog.greeden.me/en/2026/01/19/what-is-hy-motion-1-0-a-deep-dive-into-tencents-open-source-model-that-generates-3d-human-motion-from-text-features-how-to-use-required-gpu-license-caveats/)
- [Kinetix Documentation](https://docs.kinetix.tech/integration/kinetix-api)
- [Kinetix Text2Emotes Announcement](https://gamesbeat.com/kinetix-launches-text-to-animation-ai-for-new-era-in-user-generated-content/)
- [Mootion API](https://www.mootion.com/api.html)
- [Rokoko Text-to-Motion](https://www.rokoko.com/products/studio/text-to-motion)
- [MoMask (GitHub)](https://github.com/EricGuo5513/momask-codes)
- [MDM (GitHub)](https://guytevet.github.io/mdm-page/)
- [MotionGPT (GitHub)](https://github.com/OpenMotionLab/MotionGPT)
- [MotionGPT3 (GitHub)](https://github.com/OpenMotionLab/MotionGPT3)
- [Awesome Text-to-Motion](https://github.com/Zilize/awesome-text-to-motion)
- [Quaternius Universal Animation Library](https://quaternius.itch.io/universal-animation-library)
- [Mesh2Motion](https://gamefromscratch.com/mesh2motion-open-source-mixamo-alternative/)
- [ComfyUI-HY-Motion1](https://github.com/jtydhr88/ComfyUI-HY-Motion1)
- [Cartwheel](https://getcartwheel.com/)
- [Cartwheel Launch (Deadline)](https://deadline.com/2025/05/ai-animation-firm-cartwheel-emerges-from-beta-pixar-jeffrey-katzenberg-1236404797/)
- [Meshy API](https://docs.meshy.ai)
- [ARP-Batch-Retargeting (SMPL to Mixamo)](https://github.com/Shimingyi/ARP-Batch-Retargeting)
- [smpl2bvh Converter](https://github.com/KosukeFukazawa/smpl2bvh)
