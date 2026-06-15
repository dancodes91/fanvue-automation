Here is a complete, production-ready `README.md` tailored for your GitHub repository. It clearly explains how your multi-scene continuous generation pipeline operates, outlines the underlying architecture, documents input parameters, and references `workflow12.json` to make it easy for your clients to understand and utilize the endpoint.

---

# Multi-Scene Continuous Video Generation Handler (Wan2.1 + ComfyUI)

A production-grade, highly optimized serverless RunPod worker designed to generate long, coherent, multi-scene videos using **Wan2.1** via a headless ComfyUI execution engine.

Traditional image-to-video pipelines reset their visual memory with every new prompt, creating isolated, fragmented clips. This worker eliminates that limitation by passing the final frame's exact multi-dimensional latent state as a visual anchor to each sequential generation. This architecture enables the generation of unified, seamless cinematic stories across extensive durations without hard-crashing VRAM allocations or jumping frames.

---

## 🗺️ System Architecture & Continuity Engine

To achieve clean continuity without overwhelming GPU memory, the handler avoids putting heavy multi-scene nodes onto a single canvas. Instead, it treats each prompt as an independent step in an ongoing iteration loop:

```
[Uploaded Image] ──> Scene 1 (WanVAE) ──> [Final Latent Cache] 
                                                  │
                                                  ▼
[Stitched MP4] <─── FFmpeg Concat <─── Scene 2 (GetHistoryTensor)

```

1. **Scene 1 (The Anchor):** The worker initializes by consuming your static source image (`image_url`) through standard `LoadImage` bindings, processing your first prompt block.
2. **Scenes 2+ (The Continuous Chain):** The Python handler dynamically intercepts the `workflow12.json` canvas. It searches for input nodes labeled `anchor_samples` and rewrites them into an explicit **`GetHistoryTensor`** pointer. This tells ComfyUI to read the server's cache memory for the previous scene's `prompt_id` and inject its final latent frames directly into the current generation block.
3. **Lossless Assembly:** Once all scene chunks finish rendering independently, the worker fires a zero-re-encode `FFmpeg` pass to stitch the resulting clips together into a fluid, artifact-free story before uploading to cold storage.

---

## 🚀 Payload Configuration & Parameter Guide

Clients can invoke the serverless worker using the following JSON payload template.

### API Payload Example

```json
{
  "input": {
    "image_url": "https://huggingface.co/buckets/KKKONNK/used123/resolve/photo_30_2026-06-05_17-45-45.jpg?download=true",
    "fps": 16,
    "frames_per_scene": 257,
    "num_scenes": 10,
    "sampling_steps": 25,
    "prompts": [
      "Scene 1: The girl goes towards the table, picks up a notebook, and draws a butterfly in it; the drawing begins to shimmer and lifts off towards the sky.",
      "Scene 2: The glowing butterfly flies around the girl in magical golden arcs.",
      "Scene 3: The butterfly lands softly on her nose, turning back into a still pencil sketch.",
      "Scene 4: She opens the book again, causing another detailed drawing to magically come to life.",
      "Scene 5: Magical swirls of energy fill the entire room with warm golden light.",
      "Scene 6: The girl dances joyfully alongside floating paper creatures in mid-air.",
      "Scene 7: The paper drawings transform into vibrant, real animals running around her room.",
      "Scene 8: She reaches out her hand and gently touches a glowing paper bird resting in the air.",
      "Scene 9: The room dissolves into a vast forest made entirely of living notebook pages.",
      "Scene 10: She closes the notebook completely, returning the room to normal as residual stardust settles."
    ],
    "negative_prompt": "blurry, static, low quality, deformed, watermark, distorted features"
  }
}

```

### Fields Profile

| Field Parameter | Target Type | Default Value | Description |
| --- | --- | --- | --- |
| `image_url` | String | *Required* | Public direct download link to the starting frame image asset. |
| `fps` | Integer | `16` | The baseline target playback rate configuration for the compiled video. |
| `frames_per_scene` | Integer | `81` | The length of each scene chunk. Maximum safe hard ceiling value is `257`. |
| `num_scenes` | Integer | `10` | The total count of sequential story steps to execute. |
| `sampling_steps` | Integer | `25` | Total scheduler execution steps assigned to the underlying diffusion model. |
| `prompts` | Array [String] | *Required* | Sequential descriptive strings guiding the progression of the video. Ensure the array length matches `num_scenes`. |
| `negative_prompt` | String | *Optional* | A list of explicit aesthetics, artifacts, or distortions to suppress across generations. |

---

## ⚡ Hardware & Deployment Requirements

Due to the deep 3D-convolution kernels (`F.conv3d`) utilized by the **WanVAE** model structure inside `workflow12.json`, specific infrastructure rules must be followed:

* **Supported GPU Hardware:** NVIDIA A100, H100, or equivalent high-footprint enterprise profiles.
* **Warning on Blackwell Hardware:** If you target Blackwell GPUs (e.g., RTX Pro 6000 Blackwell), your environment must be updated to a PyTorch image compiled with **CUDA 13.0 or higher** to prevent native kernel image execution crashes (`cudaErrorNoKernelImageForDevice`).
* **VRAM Allocations:** Processing long `257` frame chunks requires a minimum of **40GB available VRAM** to maintain the generation pipeline smoothly.

---

## 📬 Response Output Schema

On successful generation, the worker returns a direct pointer to the stitched output along with diagnostic calculation matrices:

```json
{
  "status": "success",
  "final_video_url": "https://yaiygjwbtzevjpxncvzu.supabase.co/storage/v1/object/public/videos/job_1781465042_final.mp4",
  "chunk_urls": [
    "https://yaiygjwbtzevjpxncvzu.supabase.co/storage/v1/object/public/videos/job_1781465042_scene01.mp4",
    "https://yaiygjwbtzevjpxncvzu.supabase.co/storage/v1/object/public/videos/job_1781465042_scene02.mp4"
  ],
  "total_scenes": 10,
  "total_frames": 2570,
  "expected_duration_seconds": 160
}

```
