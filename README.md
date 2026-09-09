# MiniMax H3 · Film OS — ComfyUI Workflows

Production-tested [ComfyUI](https://github.com/comfyanonymous/ComfyUI) workflow collection for **MiniMax H3** video generation: reference-to-video (Ref2VA), temporal-chunk long-shot sampling, two-pass (双采) latent upscaling, synchronized audio, and a final **NVIDIA RTX Video Super Resolution** finishing stage.

> Workflows in this repo were battle-tested on a real AI-video production pipeline (character/scene consistency, action choreography, 720P–1080P delivery). Filenames keep their original Chinese names to match the ComfyUI workflow folder.

---

## Workflows

| File | Pipeline | Target output |
|---|---|---|
| [`workflows/Nvdia超分-720P-参考生双采.json`](workflows/Nvdia%E8%B6%85%E5%88%86-720P-%E5%8F%82%E8%80%83%E7%94%9F%E5%8F%8C%E9%87%87.json) | Ref2VA two-pass sampling → 3D latent upscale ×1.3 → RTX VSR ×2 ULTRA | 720P base → 2× RTX upscaled MP4 **with audio** |

### Featured: RTX VSR Upscale · Ref2VA Two-Pass (720P)

`Nvdia超分-720P-参考生双采` — 42 nodes / 52 links. Flow:

```
reference image(s) + text prompt
        │
        ▼
 MiniMaxH3ReferenceToVideo          ← REF2VA conditioning (spatial reference, no ghosting)
 + H3SigmaRefiner / MiniMaxH3SigmaShift (video 12 / audio 3)
        │
        ▼
 JR_H3_TemporalChunkSampler         ← Pass 1, ~0.6 MP latent
   (Hard AV Latent Prefix, 5.875s / 141-frame chunks)
        │
        ▼
 MinimaxH3LatentUpscaler3D ×1.3     ← Pass 2 re-sampling at higher resolution
        │
        ▼
 MiniMaxH3AVDecodeT8 + LTXV *AVLatent merge   ← synchronized video+audio decode
        │
        ▼
 RTXVideoSuperResolution ×2 ULTRA   ← NVIDIA RTX Video Super Resolution (final polish)
        │
        ▼
 VHS_VideoCombine → MP4 (h264, crf 19, 24 fps)
```

**Included optional LoRA stack** (set strengths in the loader nodes, or mute them for vanilla model behavior):

| LoRA | Strength | Purpose |
|---|---|---|
| `minimax_h3_fl2v_turbo_4step_v0.1_768p_sla` | 1.0 | 4-step turbo speed-up (keep ON) |
| `Minimax_H3_movieV0.1` | 0.5 | cinematic look |
| `H3_Combat_V2` | 0.5 | fight choreography |
| `动作i连续性修复LORA` | 0.5 | motion continuity fix |
| `Bunny_weapon_combatV1` | 0.6 | weapon combat |

**Built-in notes** (open the canvas — there are yellow reference cards):
- Two-pass recipes: pass1 `0.4 MP` + latent scale `1.5` → 720P (dialogue); pass1 `0.5` + scale `2.0` → 1080P. Shipped config: `0.6 MP` + scale `1.3` (action preset).
- The `3D Latent Upscaler` multiplier is exactly the pass-2 upscale factor.
- Frame count is computed from a duration input via `ComfyMathExpression` (`24 fps`, aligned to the model's 17-frame block) — edit the duration, not the frames.

---

## Requirements

### Custom nodes
Install via [ComfyUI-Manager](https://github.com/ltdrdata/ComfyUI-Manager), then restart:

| Pack | Provides |
|---|---|
| **ComfyUI core ≥ 0.33** | `MiniMaxH3ReferenceToVideo`, `MiniMaxH3SigmaShift`, `ModelAttentionBackend`, `SplitSigmas`, … |
| `comfyui_nvidia_rtx_nodes` | `RTXVideoSuperResolution` (final 2× stage) |
| `LBH-123-AI/Comfyui_Minimax_h3_latent_Upscaler` | `MinimaxH3LatentUpscaler3D` (pass-2 latent upscale) |
| `yichengup/ComfyUI-YCNodes-MiniMax-H3` | `H3SigmaRefiner` |
| `minimax-h3-audio-t8` | `MiniMaxH3AVDecodeT8` (AV decode) |
| `ComfyUI-MiniMaxH3Node` (JR) | `JR_H3_TemporalChunkSampler` (long-shot chunk sampling) |
| `comfyui-videohelpersuite` | `VHS_VideoCombine` |
| `comfyui-kjnodes` | `ModelPreviewOverrideKJ` (live preview) |
| `ComfyUI_Comfyroll_CustomNodes` | `CR Text` (prompt block) |
| `comfyui-easy-use` | `easy showAnything` |
| `rgthree-comfy` | `Label` (canvas headings) |
| `ComfyTV` *(optional)* | `ImageStage` / `ImagePickerStage` UI nodes — mute & use plain `LoadImage` if you don't run ComfyTV |
| `sol-attn` / KJ sage patch *(optional, muted)* | experimental attention speed-ups — shipped **muted** |

### Models

| File | Folder | Source |
|---|---|---|
| `DasiwaMinimaxH3_dasiwaREF2VAHybridV1.safetensors` | `models/diffusion_models/` | community (Dasiwa) REF2VA hybrid — search on Hugging Face |
| `qwen3vl_32b_minimax_h3_int8_convrot.safetensors` | `models/text_encoders/` | [Comfy-Org/MiniMax-H3](https://huggingface.co/Comfy-Org/MiniMax-H3/tree/main/text_encoders) (int8 variant; type `minimax`) |
| `minimax_h3_video_vae_fp16.safetensors` | `models/vae/` | [Comfy-Org/MiniMax-H3](https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/vae/minimax_h3_video_vae_fp16.safetensors) |
| `minimax_h3_audio_vae_fp32.safetensors` | `models/vae/` | [Comfy-Org/MiniMax-H3](https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/vae/minimax_h3_audio_vae_fp32.safetensors) |
| `minimax_h3_latent_upscaler_3d_fp16.safetensors` | per upscaler-node README | H3 3D latent upscaler |
| `taeh3.safetensors` | `models/vae/` | lightweight preview VAE |
| LoRAs (table above) | `models/loras/` | community packs — the 4 style/combat ones are optional |

### Hardware
- **VRAM:** runs on 12 GB-class cards (tested on RTX 5070). Keep single shots **≤ 10 s** at 720P two-pass; 5 s chunks are the sweet spot (the JR sampler splits longer durations automatically).
- **RTX VSR:** requires an NVIDIA RTX GPU on a recent driver. No RTX card? **Mute node `327`** — the rest of the chain still outputs the pass-2 resolution directly.

---

## Quick start
1. Install the custom node packs above, restart ComfyUI.
2. Download the models into the folders listed.
3. Open the JSON in ComfyUI (drag & drop).
4. Replace the example prompt in the **`CR Text`** block and plug your reference images into the `LoadImage` nodes — use **≥ 1080P references**: the two-pass chain compresses detail, low-res references show up soft in the final render.
5. Set your shot duration (frames auto-compute), then Queue Prompt. Pass 1 → 2 ≈ 10–12 min on a 12 GB card per 5–6 s shot.

## License
Released under **CC BY 4.0** — attribution appreciated if you share renders or derivative workflows.
