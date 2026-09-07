---
name: codex-imagegen
description: Generate or edit raster images using Codex and its built-in imagegen capability. Use when the user asks to create, generate, draw, render, visualize, or edit an image, illustration, photo, texture, mockup, game asset, icon, or other bitmap asset.
allowed-tools: Bash
license: MIT
---

# Image Generation via Codex

Generate or edit images by delegating the task to the locally authenticated Codex CLI.

Do not generate SVG, HTML, CSS, or placeholder artwork instead of an actual raster image.

This skill runs entirely through Bash by design. Inspect the project with `ls` and `find` rather than reaching for file-reading tools.

## Prerequisites

Before doing anything else, confirm that Codex is available and how it is authenticated:

```bash
command -v codex && codex --version && codex login status
```

- If `codex` is not found, stop and tell the user the Codex CLI is not installed. Do not fall back to an API-key path or to generating SVG/HTML.
- `codex login status` must report `Logged in using ChatGPT`. That is the subscription-backed path this skill is built around.
- If it reports API-key authentication instead, **stop and ask the user before continuing**. Generating in that mode bills their API account. Do not proceed on your own judgement, and never run `codex login --with-api-key` for them.
- The flag syntax below was verified against codex-cli 0.150.1. On any other version, check `codex exec --help` before invoking rather than assuming it still applies.

## Non-negotiables

1. Use Codex's built-in `$imagegen` skill, which drives the built-in `image_gen` tool.
2. Stay on the subscription-backed built-in path, confirmed by `codex login status`.
3. Do NOT use `OPENAI_API_KEY`, and do NOT use the `scripts/image_gen.py` CLI fallback that Codex's own skill offers as a secondary path. That fallback is where `gpt-image-2` and `gpt-image-1.5` live; both are out of scope here.
4. One requested asset means one Codex call. For several distinct images, issue a separate call per image rather than asking for variants of a single prompt.
5. Verify every path yourself: reference images before attaching them, and the delivered file after generation. See Validation.

## Output path

This skill has no fixed destination convention: choose the location from the project structure, the image's intended use, and the user's request. (Codex itself does have a default staging directory — see Where Codex actually saves the image below.)

If the user explicitly specifies an output path or filename, use it, and preserve a supplied filename when practical.

Otherwise, before generating:

1. Inspect the current working directory and relevant project structure.
2. Reuse an existing appropriate directory when one exists — for example `assets/`, `assets/images/`, `public/`, `public/images/`, `src/assets/`, `images/`, `docs/images/`, or `output/`.
3. Match the image's intended use to the project's existing conventions:
   - application assets go with existing application assets
   - web-served images go in the project's existing public or static asset location
   - documentation images go near existing documentation assets
   - generated deliverables follow the project's existing output convention
4. Do not create a new top-level directory unless necessary.
5. For temporary or exploratory generations with no obvious destination, pick a reasonable project-local temporary location without establishing a permanent convention.
6. Choose a short, descriptive filename when the user does not provide one.

Do not ask the user to choose an output directory when the project structure provides enough context to decide.

### Destination preflight

Once the path is chosen, and before invoking Codex:

```bash
mkdir -p "$(dirname "<OUTPUT_PATH>")"
test -e "<OUTPUT_PATH>" && echo "EXISTS: <OUTPUT_PATH>"
```

- Create the parent directory. Codex will not create it for you.
- If something is already at that path, do not overwrite it unless the user asked for replacement. Pick a sibling versioned name instead, such as `hero-v2.png` or `item-icon-edited.png`.
- Make sure Codex can write there. Under `-s workspace-write` only the working root is writable; see Working root below for destinations outside it.

### Where Codex actually saves the image

The built-in `image_gen` tool does not take a destination argument. Codex saves what it generates under `$CODEX_HOME/generated_images/...`, where `$CODEX_HOME` defaults to `~/.codex`.

So do not ask for an output path *parameter*. Instruct Codex to generate first and then **move or copy** the selected output to the path you chose. An image left only under `$CODEX_HOME` has not been delivered.

## Generation workflow

Determine from the request:

- what image should be generated
- its intended use
- aspect ratio or dimensions, if specified
- whether transparency is requested
- desired filename and output path, if specified
- any reference images supplied by the user
- how many distinct images are wanted

Inspect the project structure, choose and preflight the destination, then delegate the generation to Codex.

Always pass the prompt through a quoted here-document:

```bash
codex exec -s workspace-write --skip-git-repo-check "$(cat <<'CODEX_PROMPT'
<PROMPT>
CODEX_PROMPT
)"
```

The quoted `<<'CODEX_PROMPT'` delimiter is mandatory:

- A double-quoted prompt makes the shell expand `$imagegen` to an empty string, silently deleting the single most important instruction in the prompt.
- A single-quoted prompt breaks apart as soon as the request contains an apostrophe (`a cat's toy`, `don't`).

A quoted here-document is literal, so both survive intact. Do not use `"..."` or `'...'` for the prompt, even for short requests.

Image generation can take minutes. If your harness imposes a command timeout, raise it to at least 10 minutes for this call — in Claude Code, set the Bash tool's `timeout` to `600000` ms. A timeout is not a generation failure: check the intended output path before reporting anything to the user.

A stalled invocation looks nothing like a slow one. If the call is still running well past the timeout, check whether Codex ever started:

```bash
ls -t "${CODEX_HOME:-$HOME/.codex}"/sessions/*/*/*/*.jsonl 2>/dev/null | head -3
```

If no session was written for this run — and neither the output file nor the `-o` message file exists — Codex never got as far as recording the prompt. That is a stuck process, not a long generation. Stop it, then report. Do not re-run blindly: a second invocation can hit the same block and bills again. Look for other `codex` processes and for stale locks under `${CODEX_HOME:-$HOME/.codex}/thread-writer-locks/`, which Codex leaves behind when an earlier run is killed mid-flight.

### Flag notes

- `--full-auto` was removed. Use `-s workspace-write` instead.
- `--approve-for-me` cannot be combined with `-s`: `error: the argument '--approve-for-me' cannot be used with '--sandbox <SANDBOX_MODE>'`. This workflow does not need it — omit it. (Appending `--help` hides the conflict, so check with a real invocation.)
- `--skip-git-repo-check` is required outside a Git repository, and harmless inside one.
- Add `-o <FILE>` to capture Codex's final message. Read it afterwards to recover the path Codex reports, instead of parsing the transcript.
- If Codex fails with an unexpected-argument error, check `codex exec --help` rather than falling back to an API-key path.

### Reference images

- Give each file its own `--image=<FILE>`, and quote the value: `--image="./My Assets/ref.png"`.
- Do NOT use `-i <FILE>` or `--image <FILE>` with a space. They are the same variadic option, and it greedily eats every bare token that follows — including your prompt. Only the `=` form leaves the prompt its own slot:

  ```
  $ codex debug prompt-input --image=/tmp/a.png ONE TWO
  error: unexpected argument 'TWO' found     # one positional slot left, as intended
  $ codex debug prompt-input --image /tmp/a.png ONE TWO
  (accepted — ONE and TWO were both swallowed as image paths)
  ```
- Check every file with `test -f` before attaching it. Do not rely on Codex to reject a missing attachment: on 0.150.1 a nonexistent `--image=` path was observed to run to completion with no warning and no error, so a mistyped path yields an image generated without the reference while still looking like a success.
- Attaching a file is not enough: the prompt must say what each one is for (style reference, composition reference, the subject to preserve).

### Working root

- `-s workspace-write` makes only the working root writable. If the destination is outside it, pass `--add-dir "<DIR>"` to add that directory without moving the root.
- Use `-C <DIR>` only when Codex should run *as if* from another directory. Whenever you pass `-C`, resolve **every** path to an absolute one first — the destination, and each `--image=` attachment. Relative paths resolve against `<DIR>` for Codex but against your own directory for your checks, so the two disagree and a successful generation looks like a failure.
- Prefer `--add-dir` when the project should stay the working root; prefer `-C` for scratch work wholly outside the project.

### Prompt contents

The prompt sent to Codex MUST explicitly instruct it to use `$imagegen`, MUST name the exact destination path, and MUST tell Codex to move or copy the generated image there.

If the user asked for a transparent background or a clean cutout, say so in the prompt, ask for actual transparency from the built-in `image_gen` tool, and require that the alpha channel be preserved when the file is written. Codex treats transparency as a distinct request — leave it out and you get an opaque background that merely looks light.

### Example

```bash
codex exec -s workspace-write --skip-git-repo-check \
  -o /tmp/codex-last-message.txt \
  --image="./refs/style.png" --image="./refs/palette.png" \
  "$(cat <<'CODEX_PROMPT'
Use the built-in $imagegen skill to generate the requested raster image.

IMPORTANT:
- Use the built-in `image_gen` tool.
- Do not use OPENAI_API_KEY.
- Do not use the `scripts/image_gen.py` CLI fallback.
- Generate the actual image rather than describing it.
- The built-in tool saves under $CODEX_HOME. After generating, move or copy the
  selected image to the destination below, and report that final path.

Destination:
./assets/images/feature-card.png

Attached references, in the order given:
1. style.png — match its rendering style, lighting and texture.
2. palette.png — take colors from it only. Ignore its composition.

Image request:
A wide product card showing a single espresso cup on a matte surface,
2:1, no text.
CODEX_PROMPT
)"
```

With no reference images, drop the `--image=` arguments and the "Attached references" block; everything else stays. The paths above are examples only — do not assume `./refs/` or `./assets/images/` exists or is appropriate.

## User prompt handling

Hosts expose this skill differently — `/codex-imagegen ...` in Claude Code, `/agent-skills:codex-imagegen ...` when installed as a plugin, `$codex-imagegen ...` in Codex. Whatever the form, treat everything after the skill name as the authoritative image request:

```
$codex-imagegen 黒背景にMac Studioが置かれた未来的なAIオフィス。16:9。文字なし。
```

Expand it into a sufficiently specific Codex request without changing the user's core visual intent. Requests arrive in any language — carry the details through faithfully rather than translating them away.

## Editing images

If the user requests an edit to an existing image:

1. Identify the source image from the user's request and the current project.
2. Decide whether the result should replace the source or be written as a new file. Unless the user explicitly requests replacement, preserve the original and create a new appropriately named file.
3. Choose and preflight the destination as above.
4. Attach the source with `--image=<SOURCE>` so Codex has the actual pixels, and still name the source and destination paths in the prompt so it knows which file on disk the attachment corresponds to.

### Example

```bash
codex exec -s workspace-write --skip-git-repo-check \
  -o /tmp/codex-last-message.txt \
  --image="./assets/source.png" \
  "$(cat <<'CODEX_PROMPT'
Use the built-in $imagegen skill to edit the attached image.

IMPORTANT:
- Use the built-in `image_gen` tool.
- Do not use OPENAI_API_KEY.
- Do not use the `scripts/image_gen.py` CLI fallback.
- Edit the actual image rather than describing the change.
- The built-in tool saves under $CODEX_HOME. After editing, move or copy the
  result to the destination below, and report that final path.

The attachment is ./assets/source.png. Edit those pixels rather than
regenerating the subject from scratch.

Destination:
./assets/source-transparent.png

Requested changes:
- remove the background
- return actual transparency and preserve the alpha channel in the saved file
- preserve the character exactly
- do not change facial features
CODEX_PROMPT
)"
```

The paths above are examples only. Choose actual paths from the current project context.

## Validation

Do not rely on Codex's textual confirmation. Check the file itself:

```bash
test -s "<OUTPUT_PATH>" && file "<OUTPUT_PATH>"
```

`test -s` requires the file to exist **and be non-empty**; `file` decodes the header. Confirm all of the following before reporting success:

- `file` names a real raster format (`PNG image data`, `JPEG image data`, `RIFF ... WebP`) — not `empty`, not `data`, not `ASCII text`.
- The reported dimensions match what was requested.
- For a transparency request, the reported color type is `RGBA` (or the format otherwise reports an alpha channel). `RGB` means the transparency was lost.

If the expected file does not exist, or fails any check above:

1. Read the final message you captured with `-o` — Codex reports the path it actually wrote.
2. Otherwise look under Codex's default image location, newest first:

   ```bash
   ls -t "${CODEX_HOME:-$HOME/.codex}"/generated_images/*/* 2>/dev/null | head
   ```

3. Move or copy a candidate to the requested destination only when it is clearly the image that was just generated.
4. Report failures accurately rather than claiming generation succeeded. A file that exists but fails the format or alpha check is a failure, not a success, and a run that produced no output, no `-o` message and no session never generated anything at all.

If Codex reports that its built-in `image_gen` capability is unavailable:

- do not silently switch to an API-key path
- report that Codex image generation is unavailable in the current Codex session
- do not charge API usage without explicit user approval
