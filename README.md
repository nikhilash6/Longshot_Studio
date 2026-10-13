# H3 Long Shot Studio

A browser studio for making long AI videos with MiniMax H3, shot by shot. It runs inside
ComfyUI at `/longshot`. Build a film from short Shots, review and approve each one, and
the Studio stitches them into one continuous take or a cut sequence. Approved work is
never thrown away.

**Generate**
- Works the same way as ComfyUI Minimax H3 workflow, using references to generate video.
- Join each Shot to its neighbours with **Continuous** (flow from previous shot), **Bridge**
  (pinned at both ends), **Context** (a new angle with the last frame of the last shot as a reference)
  or **Cut**(a completely new shot with the references).
- Frame anchors: set first or last-frame images at any join.
- Multi-take renders with seed control, a render queue across different projects, and automatic
  recovery when ComfyUI runs out of memory.
- Optional song-driven lip sync and voice reference.

**Edit**
- A timeline with one block per Shot: switch takes instantly, stitch the seam when generating continuous shots, approve
  or reroll any Shot **in place** while its neighbours keep their takes.
- Insert, reorder, bypass or remove Shots anywhere, including before Shot 1.
- Drop in video clips as Shots, bridged into the generated footage or hard-cut.
- Take history for every Shot. Nothing samples again unless it has to.

**Deliver**
- Export approved clips (plain or RTX Super Resolution upscaled, incremental).
- Upscale the final film in chunks with flat RAM, with an optional ProRes 422 HQ master.
- Export or import a whole project as one .zip.

<img width="2166" height="1493" alt="image" src="https://github.com/user-attachments/assets/543bc5fd-05bf-45dd-9557-f36327e8c935" />


## What's new in 0.8.3

Needs H3 Long Shot **1.6.9 or later** (unchanged). Replace the folder, restart ComfyUI, then
hard-refresh the Studio tab.

**Join chip**
1. Every Shot has a join chip; the green ▶ renders the Shot the way the chip says.
2. The chip offers what the approved Shots next to it allow: none → Cut; before → Cut, Context, Continuous;
   after → Cut, Continuous; both sides → Cut, Context, Bridge.
3. Continuous with an approved Shot before it continues from it; with one after it, flows into it.
   Bridge is pinned at both ends. The Shots around it keep their takes.
4. The yellow ▶ (another take) uses the same chip choice.
5. The separate Bridge button is gone: the chip does it.
6. added audio reference cards


## Requirements

- A ComfyUI version with native MiniMax H3 support, and the MiniMax H3 model files.
- These custom node packs. The Studio checks for them and names any that are missing.

| Pack | Needed for |
|---|---|
| [H3 Prompt Compiler](https://github.com/r34vtraining/H3_Prompt_Compiler) | Shot, Subject, Ref Prompt Builder r2v (always) |
| [H3 Long Shot](https://github.com/r34vtraining/H3_Longshot) **1.6.9 or later** | rendering; adds the timeline mode (takes, locks, pins), Clip Shots, and the Take, Upscale Chunk and Save Audio nodes the Studio uses (always) |
| [ComfyUI-KJNodes](https://github.com/kijai/ComfyUI-KJNodes) | resizing references, optional Sage attention (always) |
| [ComfyUI-VideoHelperSuite](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite) | saving the video, loading audio (always) |
| [ComfyUI-MiniMax-H3-Turbo](https://github.com/Larryvrh/ComfyUI-MiniMax-H3-Turbo) | only when Turbo is on |
| [ComfyUI-MelBandRoFormer](https://github.com/kijai/ComfyUI-MelBandRoFormer) | only for the voice-reference audio route |
| [ComfyUI-NVIDIA-RTX-VSR-Pro](https://github.com/whmc76/ComfyUI-NVIDIA-RTX-VSR-Pro) | only for RTX upscaling (needs an RTX GPU) |

The default scheduler, `beta57`, comes from [RES4LYF](https://github.com/ClownsharkBatwing/RES4LYF).
If you don't have it, pick another scheduler under **Advanced**.

## Install

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/r34vtraining/H3_Longshot_Studio
git clone https://github.com/r34vtraining/H3_Prompt_Compiler
git clone https://github.com/r34vtraining/H3_Longshot
git clone https://github.com/kijai/ComfyUI-KJNodes
git clone https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite
```

Skip any you already have. If you already have H3 Long Shot, update it to 1.6.8 or
later with `git pull` in its folder.

Optional packs:

```bash
git clone https://github.com/Larryvrh/ComfyUI-MiniMax-H3-Turbo      # Turbo LoRA
git clone https://github.com/kijai/ComfyUI-MelBandRoFormer          # voice-reference route
git clone https://github.com/whmc76/ComfyUI-NVIDIA-RTX-VSR-Pro      # RTX upscaling
```

KJNodes, VideoHelperSuite and some optional packs have Python requirements. Install
them with ComfyUI's Python from each pack's folder:

```bash
python -m pip install -r requirements.txt
```

For the Windows portable build, run this from the `ComfyUI_windows_portable` folder instead:

```bat
python_embeded\python.exe -m pip install -r ComfyUI\custom_nodes\ComfyUI-KJNodes\requirements.txt
```

The Studio itself has no extra dependencies.

Restart ComfyUI and open **http://127.0.0.1:8188/longshot**.

## Getting started

1. The first time you open the Studio it makes an empty **Untitled** project. Rename it
   from the project menu (click the name at the top).
2. In **Settings**, check the model, text encoder, VAEs and Turbo LoRA. The Studio
   fills any it can identify from your model folders; choose the rest yourself.
3. Drop reference images on **Cast & Scenes** and give each a label (for example
   `<hero>`, `<scene1>`) and a short description.
4. Fill in **Style & Sound**, then write your Shots, referring to references by label.
5. Press **▶ Render Shot 1**, review it, then **✓ Continue** or **⟳ Reroll**.

Updating: run `git pull` in the Studio's folder (and in H3 Long Shot's), then restart
ComfyUI and hard-refresh the page (Ctrl+Shift+R).

## How it works

- `POST /longshot/build` turns the project into a ComfyUI API prompt (`graph_builder.py`,
  pure Python). Each render queues the chain of active Shots up to the target. Long Shot reuses every
  unchanged earlier segment and samples only what changed.
- The browser queues that prompt through ComfyUI's own `/prompt`, follows progress on `/ws`
  (including Long Shot's per-segment `mmh3.longshot` events), and reads the plan and the video
  from `/history`.
- Projects are saved as `ComfyUI/user/default/longshot-studio/projects/<slug>.json` (see *Projects*).
- Videos are saved to `ComfyUI/output/longshot/<slug>/videos/<project>_00001-audio.mp4`.
  Settings decides whether the first-frame PNG (`<project>_00001.png`) and the silent copy
  (`<project>_00001.mp4`) are kept; both are deleted after each render by default. Video Combine uses NVENC
  when your ffmpeg can actually run it (checked with a one-frame test encode), and the
  software h264 encoder otherwise.
- The size uses the same rule as ComfyUI's ResolutionSelector and MiniMax's size table
  (1 MP = 1024 × 1024 px, multiples of 32). For example, 0.6 MP at 16:9 gives 1056 × 608.
- Model names saved on Windows (`H3\x`) still match on Linux (`H3/x`), and the other way round.

### Layout

- **Top bar:** the project menu, then **RAM / GPU / VRAM** of the ComfyUI machine
  (refreshed every 2 s; amber above 90%; GPU load needs an NVIDIA driver),
  **Restart** and the connection light.
- **Decision row** under the viewer: **✓ Approve** (25%), the main action (50%) and
  **⟳ Reroll** (25%). Approve only approves. The main button is **✓ Approve & render
  Shot N**; with nothing left to render it becomes **Update preview** or **Upscale
  final**. Before anything is under review, Approve and Reroll are greyed out.
- **Settings** starts with **Seed · fixed**, then steps, size, RTX, reference size and
  the saved takes. **Advanced** holds the model files, Turbo and the LoRAs, then the
  sampling options. If a model file is missing, rendering stops, opens Advanced and
  marks the field.

### Model files

Advanced shows the **Model**, **Text encoder**, **Video VAE** and **Audio VAE** at the
top, each with **Browse…** (so do the Turbo LoRA, the LoRAs, and the vocal separation
model when Voice reference is on).

- **Browse…** lists every file ComfyUI can see for that slot, with a search box, the
  subfolder and size of each file, and the folders ComfyUI looks in.
- **Add a folder…** (from the machine running ComfyUI) picks a folder anywhere on
  that machine. ComfyUI lists its files straight away, no restart needed, and the
  Studio adds it again every time ComfyUI starts. The list is kept in
  `ComfyUI/user/default/longshot-studio/model_folders.json`. Folders added here can be
  removed here; folders from ComfyUI's `extra_model_paths.yaml` stay as they are.
- **Projects move between machines.** A model saved as `H3\model.safetensors` is found
  as `model.safetensors` (or in any other subfolder) on a machine that keeps it
  elsewhere, as long as only one file has that name. If a file can't be found, the
  field shows *(missing)* and rendering stops with a message naming it.

### Projects

- **Rename** moves the project's folders with it: `input/longshot/<slug>` and
  `output/longshot/<slug>` (takes and videos) take the new name, and every reference is updated,
  so nothing re-renders. It waits until ComfyUI's queue is empty, and never merges
  into a folder that already exists (it picks `name-2`).
- **Export…** (Project menu, on the ComfyUI machine) saves one `.zip` with the
  project and every file it uses: reference images, the song and video clips.
  **Include takes** is on by default, so approved Shots open already rendered with
  their take history. **Include videos** (off by default) adds every video in the
  project's videos folder. Model files aren't included.
- **Import…** (or drop the `.zip` anywhere on the page) makes a new project: files go
  into its own folders and every reference is relinked. It never overwrites a project.

- **Slug.** Each project has a fixed slug: its name lower-cased, with other characters
  turned into `-` (for example `my-chase-scene` for "My chase scene"). Names with no Latin letters or
  digits get `project-<hash>`. **Rename** changes only the display name, so the
  project's takes and videos (`output/longshot/<slug>/`) and input folder
  (`input/longshot/<slug>`) stay attached.
- **Menu.** Click the project name for **New project**, **Open…** (with the last
  video's first frame, the save time and progress), **Save** (Ctrl/⌘ S), **Save as…**,
  **Rename…**, **Duplicate** and **Delete…**. Delete can also remove the saved takes
  and the input folder; both are unticked by default, and both only work from the
  ComfyUI machine.
- **Autosave.** The project saves 2 s after any change, and right away when a render is
  queued or finishes. The indicator shows "Saved · 21:34", "Saving…" or
  "Unsaved changes". The last project reopens on load.
- **Two tabs.** If another tab saved the project since this one loaded it, saving stops
  and asks **Reload** or **Keep mine**.

### References and audio

- **What's saved.** Each reference saves its file name and subfolder (as ComfyUI has
  them), the file's size and sha256, and its original name. Audio saves the same, plus
  start, length and the three routes.
- **Where files can come from.** The file pickers list this project's input folder
  (`input/longshot/<slug>`) first, then the input folder itself.
- **Reopen checks.** On opening, every file is checked inside the input folder:
  - **ok**: shown normally.
  - **Changed since saved**: amber badge. The file was overwritten, so every Shot
    renders again.
  - **Missing**: red placeholder with **Relink…** and **Bypass**. Rendering waits until
    every active reference is found or bypassed. A missing song turns off only the
    audio routes for that render.
- **Missing files find themselves.** Copy a missing file back into the project's
  input folder (or the input folder) and switch back to the page: the Studio relinks
  it. The same contents under any name relink silently, with nothing to re-render;
  a file that only has the same name asks first, since every Shot would re-render.
- **Relinking costs nothing.** Picking a file with the same contents, even from another
  folder, restores it silently with nothing to re-render.
- **Drag and drop.** Drop images on **Add reference** (or anywhere in Cast & Scenes) to
  add one reference per image. A single new image opens the editor so you can name it.
  Drop an image on a card to replace that card's picture. Clicking an empty thumbnail
  opens a file picker.
  - Files go to `input/longshot/<slug>/` through `POST /longshot/upload`, from any device
    that can reach ComfyUI. The size limit is ComfyUI's `--max-upload-size`.
  - Uploads never overwrite. A file with the same contents is reused, so dropping the
    same picture again re-renders nothing. A different picture with an existing name
    is saved as `name (2).png`.
  - Only images are taken (PNG, JPG, WebP, BMP, GIF, TIFF), and each is checked as a
    readable image.
- **Audio upload.** Drop a song anywhere on the Audio panel (even closed), or use
  **Upload…** next to the file list. It goes into the project's input folder and
  becomes the project's audio.
- **Audio preview.** The play button next to Length plays the clip the render will use:
  from Start, for Length seconds (0 plays to the end). It plays at the viewer's volume
  level, stops by itself at the end of the clip, and stops when the video plays.
- **Subfolders work for audio.** `VHS_LoadAudioUpload` validates the file by path, not
  by its dropdown, so `input/longshot/<slug>/song.mp3` loads. Images and audio in a
  project subfolder are tested through ComfyUI's validator and executor
  (`test_references_and_audio_load_from_a_project_subfolder`).

### Takes

Every Shot you render is kept as a **take**: a file in
`output/longshot/<slug>/takes/` named `<shot>__<seed>__<id>.safetensors` (about
4–5 MB for a 5 s Shot at 0.6 MP). Approved Shots, and the Shot under review, load
their take instead of sampling.

- **Crash or restart?** Reopen the project. A free dry run (the model doesn't load)
  shows every finished Shot as **take** in the Plan, and Continue picks up where you
  left off.
- **Projects from earlier versions** keep their saved segments: the first render
  adopts them as takes, so nothing re-renders.
- Settings shows **Saved takes: N files · X MB** and **Delete all takes** (from the
  ComfyUI machine only).

### Reviewing and rerolling

- **Seed** (Settings) is the next seed. With **Increment**, each Shot that renders takes it
  and it goes up by one (renders, rerolls, bridges, Multi-take), like ComfyUI's control
  after generate. With **Fixed** it stays, every Shot on auto uses it, and rerolls pick
  random seeds. A Shot left on auto stays on auto: its row shows the seed its take used
  ("auto · 1722") and its next render takes a new one. A seed typed into a Shot is fixed.
  Clear render keeps the seed for one more render, so the same take comes back.
- **Reroll in place.** ⟳ on an approved Shot gives it a new take with the same length,
  pinned to the Shot before it and to the Shots after it, which stay exactly as they
  are. One render.
  - The dialog has **Also re-render the Shots after it** for when the change should
    ripple: those Shots go back to Queued, keep their text and seeds, and are marked
    "was ✓"; **Re-render through Shot N** then brings them back in one queue.
  - Editing a rendered Shot's text, seconds or seed also re-renders it in place; the
    toast offers to re-render the Shots after it too.
- **Take history.** A Shot with more than one take shows **Take 2 of 4 ◀ ▶** beside
  Reroll (and in its card). Switching takes is instant: the take is a file, so only
  the preview is rebuilt. Click the label for every take, to use one or delete
  ones you don't need (deleting works from the ComfyUI machine only).
- **Add Shots anywhere.** **Add shot before Shot 1** sits above the list, and an open
  Shot card has **⊕ Shot before** / **⊕ Shot after**. A new Shot renders once, pinned
  to the Shots around it; everything else keeps its take.
- **Hard cuts.** If a Shot no longer follows the take before it (say you removed or
  bypassed the Shot in between), its card shows **↯ hard cut**, and the Plan says so.
  Set the join chip of the Shot to render to Continuous or Bridge and press ▶ to join
  them; the Shots around it keep their takes. After removing, bypassing or switching takes,
  **▶ Update preview** rebuilds the video without sampling anything.
- **Lip sync.** Anything that moves later Shots against the song (inserting, removing,
  bypassing, changing a Shot's length) re-renders those Shots so they stay in sync.
- **References** open in an editor dialog when you click the label, the description or
  ✎. It shows the image large, the label, and an auto-growing description box.
  - Ctrl/⌘ Enter saves and Esc cancels.
  - Clicking outside asks before discarding your edits.
  - Saving asks first when rendered Shots would re-render.
- **Clicking a thumbnail** opens the lightbox: scroll to zoom, drag to pan, ←/→ to move
  between references, Esc to close.
- **Bigger viewer.**
  - **Theater** (▭ button or `T`) spans the page, with the decision panel beside it.
  - **Full screen** (⛶ button or `F`) puts the video, every control and the decision panel
    on screen, with the controls in a translucent bar along the bottom that sits in the
    letterbox space first.
  - Esc or the same button exits. Space and ←/→ (frame step) work in every view.
- **Volume.** The speaker button mutes and unmutes, the slider sets the level, and `M`
  toggles mute. Both are in every view, and the browser remembers the level and mute
  state between sessions.

### Clip Shots

A Shot can be a real video clip instead of a prompt: footage, a stock clip, or
anything rendered elsewhere. The generated Shots around it lead into and out of it.

- **Adding one:** drop a video on the Shots list (it goes after the Shot you drop it
  on), **⊕ Clip after** or **Replace with clip** in an open Shot, or **＋ Clip before
  Shot 1**. Replace keeps the old Shot, bypassed, so you can switch back, and matches
  its length.
- **The card** shows a filmstrip, the video (pick another or **Upload…**), **Start**
  and **Length** (snapped to a valid length; it shows the result), and **Sound**: the
  clip's own or mute. Clips are resized and centre-cropped to the film's size, at 24 fps.
- **Joins:** the marker between a clip and its neighbour is **Bridge ↔** (the generated
  Shot re-renders to flow into or out of the clip; the default) or **Cut |** (it stays
  as it is: a hard cut). Click it to switch. Two clips together always cut.
- **What re-renders:** adding a clip, or changing its trim, length, sound or file,
  re-renders only the bridged Shots next to it. Replacing Shot 4 of 6 re-renders
  Shots 3 and 5; the rest keep their takes.
- **The shared zone:** each join shares 0.92 s (at the default overlap): on a bridge
  the Shot before the clip arrives there; on a cut those first 0.92 s of the clip are
  hidden. The card says which.
- **Final video:** Settings → **Final video: original clip pixels** puts each clip's
  original frames back when you **Upscale final video** (sharper clips, possible faint
  seam at their edges). Otherwise clips come out of the decoder like everything else.
- The viewer marks clip spans in blue. Clip files are listed and relinked like other
  references, and travel with exports.
- ComfyUI's upload limit (`--max-upload-size`, 100 MB by default) applies to dropped
  videos; for bigger files, copy them into the project's input folder and pick them
  on the card.

### RTX Super Resolution (upscale previews / upscale final)

Two controls use NVIDIA's RTX Video Super Resolution node (`RTXVideoSuperResolution`,
"scale by multiplier"), which runs between VAE Decode and Video Combine. Neither ever re-renders a Shot: the upscale runs after Long Shot,
so segments are reused and only the decode, upscale and save run again.

- **Upscale previews** (switch in Settings, under the resolution settings). When it's
  on, every render is upscaled. It gets slower as the chain grows, so leave it off
  while you review.
- **⤢ Upscale final video** (decision panel, once every Shot is approved). It always
  upscales, with the scale and quality set next to the switch, and saves as
  `longshot/<slug>/videos/<project>_final_#####-audio.mp4`. After that, the panel shows
  the saved file name and offers "again".
- **ProRes master** (checkbox under the upscale settings): the final upscale also writes
  `<project>_final_master_#####-audio.mov`, 10-bit ProRes 422 HQ with PCM sound and the
  workflow metadata, from the same frames. Expect large files (about 1.3 GB per minute
  at 1080p). If ComfyUI hasn't loaded the Studio's ProRes format yet (restart), Video
  Helper Suite's own ProRes is used and the Studio says so.
- **Scale** is 1.5× or 2×. Higher factors mostly run out of memory on long chains.
- **RAM.** ComfyUI holds every frame in system RAM as float32, before and after the
  upscale. A 35 s chain at 0.6 MP needs about 20 GB at 1.5× and 30 GB at 2×. The
  Settings row shows the estimate for your current chain.
- **Needs** ComfyUI-NVIDIA-RTX-VSR-Pro and an RTX GPU. Without the pack, both controls are
  greyed out.

### Restarting ComfyUI

**Restart** (top bar, left of the status light) restarts ComfyUI after asking first.

- ComfyUI relaunches itself with the same command line, the way ComfyUI-Manager does it,
  minus `--windows-standalone-build` so no extra browser tab opens.
- The page shows "Restarting ComfyUI…", reconnects on its own, and re-checks saved
  segments.
- A running render stops. Finished segments are on disk, so nothing already rendered
  is lost.
- It works from any device that can reach ComfyUI, including over Tailscale.

### What re-renders what

- **Editing a Shot's text, seconds or seed** re-renders that Shot in place. The Shots
  after it keep their takes (or re-render too, if you ask).
- **Bypassing, removing or inserting a Shot** re-renders nothing by itself; the Shots
  renumber and keep their takes. With lip sync on, the Shots after it re-render to stay
  in sync with the song.
- **Cast & Scenes, Style & Sound, Settings, and the lip-sync or voice audio routes** are
  shared by every Shot. Changing them re-renders everything from Shot 1. Earlier approvals
  are kept as a "was ✓" flag.
- **Song in final video** only changes what Video Combine muxes, so nothing re-renders.

### Quality notes for pinned Shots

- A Shot pinned at both ends (an in-place reroll, or a Shot inserted between
  rendered ones) has to travel from its start to a fixed end state. Give it at
  least 5 s and write the arrival into its text ("…ends with her at the doorway,
  facing screen left"). If the states are too different, the last second can
  visibly morph.
- End pins are a newer guide shape than first/last frames; check them early on
  real renders. The fl2va model is expected to behave best.
- The first frames after a pinned join can decode very slightly differently; this
  is expected to be invisible.

### Standalone Shots and bridging

On long locked-off takes, small drifts add up from Shot to Shot. To keep drift from
travelling, render every other Shot fresh and let the Shots between them bridge:

1. **▶ Render odd Shots** renders Shots 1, 3, 5… each on its own, straight from the
   references (or press **▶** on single Shots).
2. **▶ Render all** renders Shots 2, 4, 6… in one run, each pinned to the ending of the
   Shot before it and the opening of the Shot after it.
3. Review them one by one with **✓ Approve & review Shot N**.

Pressing the main button instead of Render all also works: it renders one gap at a time.
For the standard workflow (every Shot continued from the one before), use **Render all**
on its own.

Drift can then build up across one Shot at most. The bridges can only join what the
standalone Shots give them, so write each standalone Shot to **open close to where the
previous Shot ends** (same framing, positions and lighting). A bridge shares the overlap
frames (Advanced → Overlap) at both ends with its neighbours, so the first fraction of a
second of each standalone Shot is where the bridge arrives.

- A standalone Shot stays locked to its take in every later render; nothing continues
  it from the left unless it renders again with its chip on Continuous or Bridge.
- Rerolling or re-rendering a standalone Shot after an edit keeps it standalone. If the
  Shot before it was already rendered, that join becomes a hard cut until you reroll the
  Shot before it (pinned to both ends, chip on Bridge).
- Changing references keeps approved standalone Shots, like any approved Shot.

### Opening folders

The **Open input folder** and **Open output folder** buttons open the folder on the machine
running ComfyUI (the output button opens the project's videos folder and selects the video). They only work when you browse from that machine (a loopback connection with
no proxy headers). From another device, such as over Tailscale, the buttons are hidden and
**Download video** appears instead.

## Tests

`tests/` runs against a real ComfyUI source tree in CPU mode, with no model weights:

- `test_graph_builder.py`: pure Python, no ComfyUI needed. Covers every toggle combination
  (the 8 audio-route combinations, bypassed cast and Shots, LoRAs on and off with and without
  Turbo, Sage missing, NVENC fallback, missing packs) and MiniMax's size table.
- `test_comfy_integration.py`:
  - ComfyUI's own `validate_prompt` accepts every variant, with the real packs loaded.
  - The full core loop (dry run, Start, Continue, Reroll, edit, bypass, audio routes) runs
    through ComfyUI's real executor. Loaders, decoders and the sampler are stand-ins; Long
    Shot, the prompt nodes, LoadImage, KJ resize and VHS are real, and VHS writes a real mp4.
  - Also covers the HTTP routes.
  - The ProRes master is written through the real VHS and checked with ffprobe (ProRes HQ,
    10-bit 4:2:2, PCM sound, workflow metadata), and the PNG / silent-video cleanup runs on
    real render output.
- `test_studio_server.py`: who counts as local, open-folder path safety, portable model names.
- `test_projects.py`:
  - slugs, including non-Latin names;
  - save → reload gives identical state;
  - the slug survives a rename;
  - two-tab conflicts;
  - missing / changed / relinked references;
  - `check-inputs` path refusal;
  - Delete and Clear touch only the chosen files;
  - videos move with a rename, export / import with and without videos, and the render
    cleanup never leaves `output/longshot` or touches the video with sound.
- `test_comfy_integration.py` also covers crash-resume. After a simulated restart,
  nothing is re-sampled and the model never loads until a new Shot renders.

```
set COMFYUI_ROOT=C:\path\to\ComfyUI
set MMH3_PROMPT_PACK=%COMFYUI_ROOT%\custom_nodes\H3_Prompt_Compiler
set MMH3_LONGSHOT_PACK=%COMFYUI_ROOT%\custom_nodes\H3_Longshot
set STUDIO_EXTRA_NODES=%COMFYUI_ROOT%\custom_nodes\ComfyUI-KJNodes;%COMFYUI_ROOT%\custom_nodes\ComfyUI-VideoHelperSuite;%COMFYUI_ROOT%\custom_nodes\ComfyUI-MiniMax-H3-Turbo;%COMFYUI_ROOT%\custom_nodes\ComfyUI-MelBandRoFormer;%COMFYUI_ROOT%\custom_nodes\ComfyUI-NVIDIA-RTX-VSR-Pro
python -m pytest tests -q
```

Use your ComfyUI's Python. For the portable build that's `python_embeded\python.exe -m pip install pytest`,
then `python_embeded\python.exe -m pytest ...`. If a pack folder isn't set, the tests that
need it are skipped and say why.
