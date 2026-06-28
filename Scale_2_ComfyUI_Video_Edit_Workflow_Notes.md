# Scale 2 ComfyUI Video Edit Workflow Notes

These notes are distilled from the attached transcript. The workflow is for video editing with Scale 2: character replacement, background replacement, localized object edits, optional mask compositing, and chunked extension for longer videos.

## Core Idea

Scale 2 edits an existing video using:

- A source video.
- A reference image, usually the first frame of the source video after editing the target subject/background into the desired look.
- Text prompts describing what is happening in the scene.
- A video mask, typically generated with SAM 3.1 Multiplex, to restrict what gets replaced.
- Optional compositing to keep untouched areas from the original video.

It is not described as a pure reference-to-video workflow. The reference image guides the edit while the source video provides motion and temporal structure.

## Reference Preparation

1. Extract or select the first frame from the source video.
2. Edit that frame into the target look with a separate image editor/model, such as Nano Banana, ChatGPT image editing, or Qwen Image.
3. Keep the pose and framing close to the source frame.
4. Use a practical Scale 2 size:
   - Landscape: `896 x 512`
   - Portrait: `512 x 896`

The transcript recommends using this edited first frame as the reference image when replacing a character or a specific region.

## Model Area

The transcript mentions this model stack:

- Scale 2 model:
  - Prefer FP8 if available.
  - Q8 is mentioned, but the presenter switches to FP8.
- LoRA:
  - `LyteX 2v mis 2v 64 rank distill auto`
  - Lower VRAM alternatives: rank `32` or `16`.
- Text encoder:
  - `umt5_xxl_fp8` / `UMT5 XXL FP8`
- VAE:
  - `Wan 2.1 VAE`

The presenter also mentions a ComfyOrg DPO LoRA that may slightly improve video edit quality, but says the difference is minor.

## Sampling Settings

Suggested baseline from the transcript:

- Sampler: `Euler`
- Steps: `8`
- Denoise: `1.0`
- Faster draft option: `4` steps
- Frame count example: `81` frames

Use lower frame counts and lower-rank LoRAs if VRAM is limited.

## Prompting

Keep prompts simple. The presenter says not to over-describe.

Use a short description of the scene/action, for example:

```text
A person walks forward through the room while raising one arm.
```

For localized edits, use separate text conditioning when possible:

- Reference prompt: describes the new subject or object.
- Mask/video prompt: describes the area to detect or replace.

Example target mask prompts:

```text
the person
the shirt
the right arm
the background
the gauntlet
```

## Masking

The transcript uses SAM 3.1 Multiplex for masking.

Typical mask flow:

1. Load source video frames.
2. Detect target region with SAM 3.1 Multiplex.
3. Feed the mask into the Scale 2 color mask / driving data node.
4. Feed pose video, mask, reference image, and conditioning into the Scale 2 video node.

Replacement mode note from the transcript:

- For background replacement, turn the placement/replacement mode off.
- For character replacement, turn the character/person replacement toggles on.

Exact toggle names may differ depending on your installed node version.

## Main Node Flow

The high-level graph described in the transcript:

```text
Source video
  -> video frames / first frame
  -> SAM 3.1 mask
  -> Scale 2 color mask driving data

Edited first frame reference
  -> Scale 2 video node

Scale 2 model + LoRA + CLIP + VAE
  -> text conditioning
  -> Scale 2 video node
  -> SamplerCustom
  -> VAE decode
  -> video combine/save
```

The key node named in the transcript is `Scale 2 Video`. It receives the source pose/video, masking data, reference image, and conditioning, then outputs latent data for `SamplerCustom`.

## Compositing For Local Edits

If only one object or body part should change, composite the generated region back over the original video.

Reason:

- Full-video sampling can alter faces, background, and other regions.
- Using the same mask after sampling lets you keep the original video everywhere except the edited region.

Flow:

```text
Original video frames
Generated video frames
SAM mask
  -> composite generated masked region over original frames
  -> final video
```

This is useful for edits like replacing only an arm, gauntlet, shirt, or small prop.

## Extending Longer Videos

The transcript describes using ComfyUI Easy Use loop nodes:

- `For Loop Start`
- `For Loop End`

Chunking idea:

1. Generate the first chunk, such as `49` or `81` frames.
2. Send generated video output into loop initial value 1.
3. Send `video_frame_offset` from the Scale 2 video node into loop initial value 2.
4. Feed previous frames back into the Scale 2 video node.
5. Feed video frame offset back into the Scale 2 video node.
6. Merge each loop output into the final video.

Loop count formula from the transcript:

```text
total_source_frames / frames_per_chunk
```

Example:

```text
300 total frames / 49 frames per chunk = about 7 loop iterations
```

This avoids loading/generating the whole video at once and helps prevent out-of-memory errors.

## Practical Defaults

Start with:

- Resolution: `896 x 512` or `512 x 896`
- Frames: `49` or `81`
- Steps: `8`
- Sampler: `Euler`
- Denoise: `1.0`
- Mask model: `SAM 3.1 Multiplex`
- Use compositing for localized edits.
- Use loop extension only after the base 5-second edit works.

## Troubleshooting

- If the whole frame changes unexpectedly, add compositing with the original video.
- If the wrong area changes, refine the mask prompt and preview the SAM mask before sampling.
- If motion drifts, make sure the reference frame matches the source frame pose and framing.
- If VRAM runs out, reduce frame count, use a lower-rank LoRA, reduce resolution, or use looped chunks.
- If character replacement is weak, improve the edited first frame and keep the subject pose closer to the source.

