# Woven's private Hanabi patch

This maintenance branch starts exactly at upstream `bevy_hanabi` 0.19.0 commit
`cf35d53d54fb973e30c635e7097ff3e6abf33099` (annotated tag `v0.19.0`; tag
object `ae3bc5fc18e89ab619cb035a0475f9b1335ce897`). The Git ancestry records that
base directly. Upstream licenses the source as MIT OR Apache-2.0; the
unmodified `LICENSE-MIT` and `LICENSE-APACHE2` files remain authoritative.

Woven carries eight narrow private seams:

1. The already allocated per-instance `properties_array_index` is copied into
   `GpuSpawnerParams` and the WGSL `Spawner`, then installed in the private
   property-expression index for init, update, and render modifiers. Property
   buffers, uploads, bindings, allocation, and lifetime remain Hanabi-owned.
2. `GpuSpawnerParams`/`Spawner` carries `simulation_paused: u32` plus three
   explicit trailing padding words. The Woven property-index seam made the row
   128 bytes; the pause seam intentionally changes it to 144 bytes, still with
   16-byte CPU/WGSL alignment. Paused init dispatches return before allocating
   particles. Paused update dispatches copy the existing alive indirection row
   to the render side, then return before age, reap, modifier, trail, beam,
   light, sprite, model, or particle-state updates. Render still observes the
   existing particles and the current property index. Resume uses the same
   instance and resumes normal init/update work.
3. `PropertyLayout` reserves each value's complete WGSL size and alignment.
   In particular, `mat3x3<f32>` occupies 48 bytes and `mat4x4<f32>` occupies
   64 bytes; the regression layout places them at offsets 0 and 48, places the
   following scalar at 112, reports 116 CPU bytes, and rounds the binding to
   128 bytes. This prevents a matrix upload from overlapping the next property.
4. `EffectMaterial` and `TextureLayout` carry separately declared texture-only
   and sampler-only bindings for Woven's public particle parameter schema.
   Render WGSL places those resources after Hanabi's ordinary paired material
   textures in the existing material bind group. `EffectSampler` is the private
   typed sampler value; imported mode follows the paired texture sampler, while
   the remaining values select engine-owned nearest/linear and
   clamp/repeat/mirror descriptors. Live image handles and sampler values are
   bind-group data and never enter shader or pipeline specialization identity.
5. `EffectPipelinePrewarm` asks extraction and queueing to specialize the same
   init, update, and routed render pipelines as a visible effect without
   drawing it. The prewarm allocation is one particle, not the authored
   capacity. `EffectPipelineStates` exposes only `Preparing`, `Ready`, or
   `Failed` per main-world entity from Hanabi's queued compute/render pipeline
   IDs and Bevy's device pipeline cache. Woven retains the accepted prewarm
   entities for device recreation and retires prior ones only after the render
   queue reports completion of their final submitted work.
6. `EffectSimulationPaused` is extracted as data for the spawner-row pause bit
   described above; it is not a pipeline key. Property uploads and render
   binding updates therefore remain live while motion, age, and emission are
   paused on the existing emitter.
7. The `woven_internal_timing` feature exposes only the hidden
   `woven_private::{ParticleGpuStage, ParticleTiming,
   ParticleTimingProvider}` seam. Hanabi requests semantic spans for uploads,
   buffer-growth copies, init-fill, init, indirect dispatch, update prefix,
   update, optional sort prefix, sort-fill dispatch, and the existing combined
   sort-fill/sort/sorted-index-copy pass. Woven owns query sets, raw indices,
   resolution, aggregation, and budgets. With this feature enabled, every
   Hanabi CPU upload and buffer-growth copy uses the existing `copy_buffer`
   compute pipeline in ordinary Play and capture. One descriptor-timestampable
   pass covers each semantic stage; capture changes only whether the provider
   returns timestamp writes. Feature-disabled Hanabi retains its upstream
   queue-write and encoder-copy paths. `ParticleRenderBatch` marks Hanabi's
   temporary draw entities so Woven can classify routed render runs without
   changing their sorted phase.
8. The utility compute shader module uses the diagnostic label
   `hanabi_shader_utils`. DXC receives shader-module labels as source filenames
   and rejects the former colon-delimited `hanabi:shader:utils` label on Windows.
   This changes no WGSL source, Bevy shader import path, pipeline key, or
   particle behavior; other labels remain intact for diagnostics.

The branch also selectively adopts two post-tag correctness fixes without the
intervening batching, storage, shader-layout, sorting, or feature rewrites:

- `9a4e95b`: `AlignedBufferVec` reports reallocation and GPU-operation bind
  groups are invalidated before they can retain a stale args buffer; and
- `5fab912`: the cached-event observer is the sole owner of event-buffer free,
  and metadata bind groups are invalidated when that buffer changes.

It also carries one Woven-authored correctness fix. In v0.19.0, image
`Modified` and `Removed` events evict only the per-image bind group cache, never
the material bind groups cached per set of image ids. Each removed image then
stays resident through its cached material bind group, and a modified image keeps
drawing its previous texture; an effect whose texture alternates between assets
retains one bind group and one GPU texture per reload. Both events now evict every
cached bind group that binds the image.

The copied `vfx_common.wgsl` mechanically normalizes its upstream CRLF line
endings while adding the matching Spawner fields. That normalization has no
semantic effect. The eight seams, two adopted fixes, and the material bind group
eviction above are the complete intentional behavior/layout delta from upstream
v0.19.0.

The graph-node test imports `Vec3` explicitly so Bevy's unrelated UI `Node`
type cannot make the graph `Node` trait ambiguous. This is test-only and has no
production behavior.

These private seams can be removed once upstream exposes equivalent render
property access, per-instance paused simulation, correct matrix property
layout behavior, independent material-resource bindings, and observable
device-pipeline prewarming. The timing seam can be removed once upstream
exposes equivalent semantic stage hooks, descriptor-timestampable uploads and
growth copies, and semantic particle-draw classification.

## Descriptor-timestampable copy proof

The H2 copy change was verified on 2026-08-28 on the Apple M4 Pro Metal
adapter. `cargo test --lib --features woven_internal_timing,3d` passed 176
tests with the physical oracle ignored by default. Running
`particle_copy_compute_passes_have_positive_physical_timestamps` explicitly
passed and required positive intervals for both the real upload queue and the
real growth-copy dispatcher. The `single_particle` and `properties`
all-features graphical tests also exited successfully.

`cargo fmt --all -- --check`, `cargo check --all-features`, `cargo check --lib
--no-default-features --features 3d`, and `cargo clippy --lib --tests --features
woven_internal_timing,3d -- -D warnings` passed. An additional all-targets
clippy probe reached the three examples that the prior H1 commit left without
the private `EffectMaterial.woven_samplers` field; that pre-existing example
failure is not part of the H2 copy-timing delta.
