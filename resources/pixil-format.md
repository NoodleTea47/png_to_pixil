# The Pixilart `.pixil` File Format

Reverse-engineered reference for the project file saved by the
[Pixilart](https://www.pixilart.com/draw) online pixel-art editor.
Pixilart publishes no specification. Everything here comes from inspecting
real files, third-party converters, and Pixilart's public help text.

**Evidence base.** Two files saved locally from Pixilart 2.7.0 in September
2026 (`resources/*.pixil`), about 80 `.pixil` files found on GitHub spanning
format versions 1, 2.6 and 2.7.0 (2017 to 2026), four open-source readers or
writers, and Pixilart's own help pages. Each claim below is tagged:

| Tag | Meaning |
|-----|---------|
| **Observed** | Seen directly in genuine Pixilart output |
| **Inferred** | Consistent with all samples but not confirmed by Pixilart |
| **Reported** | Stated by a third-party source, not independently verified |
| **Unknown** | Field exists but its meaning is not established |

---

## 1. Overview

A `.pixil` file is a single UTF-8 JSON document. There is no binary framing,
compression, or checksum. The pixel data for every layer is an ordinary
**8-bit RGBA PNG**, base64-encoded inside a `data:` URI whose MIME prefix has
been deliberately mangled (see section 6).

The structure is a three-level tree:

```
document
├── metadata (app, version, canvas size, palettes, timestamps)
├── frames[]                 ← animation frames, in playback order
│   ├── metadata (speed in ms, selected layer, ...)
│   ├── layers[]             ← bottom to top
│   │   ├── src              ← full-canvas PNG as data URI
│   │   └── options          ← blend, lock, filters
│   └── preview              ← flattened PNG of this frame
└── preview                  ← flattened thumbnail of the document
```

Every layer PNG has the full canvas dimensions. Layers carry no offset,
crop, or tiling information. **Observed** across all samples.

---

## 2. Top-level fields (version 2.7.0)

Field order as written by Pixilart. Types shown are the ones observed;
see section 9 for type inconsistencies.

| Field | Type | Meaning | Tag |
|-------|------|---------|-----|
| `application` | `"pixil"` | Constant | Observed |
| `type` | `".pixil"` | Constant. Added in 2.7.0 | Observed |
| `version` | `"2.7.0"` | Format version. String in 2.7.0, number in older versions | Observed |
| `website` | `"pixilart.com"` | Constant | Observed |
| `author` | `"https://www.pixilart.com"` | Constant | Observed |
| `contact` | `"support@pixilart.com"` | Constant | Observed |
| `width`, `height` | int or numeric string | Canvas size in pixels | Observed |
| `colors` | object | Palettes, keyed by palette name (section 3) | Observed |
| `colorSelected` | string | Key of the active palette in `colors`. One sample holds a `#rrggbb` colour instead | Observed |
| `frames` | array | Animation frames (section 4). Always at least one | Observed |
| `currentFrame` | int (string in v1) | Index of the frame open in the editor | Observed |
| `speed` | int | Default frame duration in ms for new frames. Always 100 in samples | Inferred |
| `name` | string | Drawing title. `"Untitled"` by default | Observed |
| `preview` | data URI | Flattened thumbnail of the whole drawing, with the junk token spliced in (section 6) | Observed |
| `previewApp` | string | Usually `""`. `"pixil"` in one 2026 file that also lacked the junk token, the newer option fields and frame dimensions, used `"Frame 1"` as a frame name and a `#rrggbb` value in `colorSelected`. That file was almost certainly written by a different client, probably the Pixilart mobile app | Inferred |
| `art_edit_id` | int | Always 0 in samples. Presumably the server-side ID of a published artwork being re-edited | Inferred |
| `palette_id` | bool | Always `false` in samples. Presumably a saved-palette reference | Inferred |
| `created_at` | int | Unix time in **milliseconds** | Observed |
| `updated_at` | int | Unix time in milliseconds | Observed |
| `id` | int | Unix-millisecond timestamp from when the drawing was created. Equal to or slightly before `created_at` | Observed |
| `persLayers` | bool | Present and `true` in some files. Relates to the "Persistent Layers" feature (section 8). Absent in most files, including new ones, so absence does not mean off | Observed |
| `isExternal` | bool | Seen in two 2024 files. Meaning unknown | Unknown |
| `edit` | `{status: bool, unqid: string}` | Seen in one local file. Probably editor session state for a re-opened drawing | Unknown |

Fields other than `frames`, `width`, `height` and the layer `src` values are
ignored by every third-party reader examined, and the synthetic test files
that omit them still claim to load in Pixilart. **Reported.**

---

## 3. Palettes: `colors`

`colors` is an object whose keys are palette names and whose values are
arrays of colour strings. **Observed.**

- Colours are six lowercase hex digits with **no** leading `#`,
  for example `"f44336"`. Version 1 files mix upper and lower case.
- Alpha is not stored in palettes.
- Built-in palettes in 2.6 and 2.7.0 are `default` (274 entries, Material
  Design colours), `simple` (84), `common` (25) and `skin tones` (18).
- Version 1 used `default colors`, `common`, `skin tones`, `metro ui` and
  `rainbow dash`.
- Users can add palettes; one file had 23 named palettes.
- The active palette grows when the user picks new colours: one local file's
  `default` palette had four extra entries appended (`00ff00`, `f44040`,
  `40ff40`, `00ff40`). **Observed.**
- `colorSelected` names the active key. Files written by converters use an
  arbitrary key such as `"palette"` and Pixilart accepts it. **Reported.**

Palettes are editor state only. They do not constrain the pixel data.

---

## 4. Frames

Each element of `frames`:

| Field | Type | Meaning | Tag |
|-------|------|---------|-----|
| `name` | string | Frame label. Usually `""`; one file used `"Frame 1"` | Observed |
| `speed` | int | Frame duration in **milliseconds**. Default 100. Pixilart's help text says "1000ms = 1 second" for frame time | Observed |
| `layers` | array | Layers, bottom to top (section 5) | Observed |
| `active` | bool | `true` on every frame in every genuine sample, including a two-frame animation. Not "current frame" | Unknown |
| `selectedLayer` | int | Index into `layers` of the layer selected in the editor | Observed |
| `unqid` | string | Random 5 to 10 character alphanumeric ID | Observed |
| `preview` | data URI | Flattened composite of the frame's visible layers. Plain mangled prefix, no junk token | Observed |
| `width`, `height` | same type as top-level | Duplicate of canvas size. One 2026 file omitted them | Observed |
| `data_id` | int | Seen in two files, value 0. Meaning unknown | Unknown |

The frame `preview` is byte-for-byte identical to the PNG Pixilart produces
when you export the drawing as PNG. Verified on both local files. **Observed.**

---

## 5. Layers

Each element of a frame's `layers`:

| Field | Type | Meaning | Tag |
|-------|------|---------|-----|
| `id` | int | Position within the frame at creation time. Zero-based in all but one sample, which started at 1 | Observed |
| `src` | data URI | Full-canvas RGBA PNG holding this layer's raw pixels (section 6) | Observed |
| `edit` | bool | `false` on untouched layers, `true` on layers that have been drawn on. Added in 2.6 | Inferred |
| `name` | string | Layer name. New drawings start with `"Background"`; added layers are `"Layer N"` | Observed |
| `opacity` | numeric **string** `"1"` (number in v1) | Layer opacity 0 to 1. Not baked into `src` | Observed |
| `active` | bool | **Layer visibility.** Hidden layers are `false` | Observed |
| `unqid` | string | Random alphanumeric ID. The same logical layer keeps the same `unqid` in every frame of an animation | Observed |
| `options` | object | Blend, locks and filters (section 5.1). Added in 2.6 | Observed |

### Visibility proof

In a seven-layer file where only one layer was `active: true`, compositing
just the active layers reproduced the stored frame `preview` exactly, while
compositing all layers did not. In a two-layer file with both active, the
composite matched only when layer index 0 was treated as the **bottom**.
**Observed.**

### 5.1 `options`

| Field | Type | Meaning | Tag |
|-------|------|---------|-----|
| `blend` | string | Canvas 2D `globalCompositeOperation` name. Only `"source-over"` (Normal) appears in samples | Observed |
| `alpha_lock` | bool | Alpha Lock: painting is restricted to already-opaque pixels. Added around 2025 | Observed, Reported |
| `locked` | bool | Layer is locked against editing | Observed |
| `filter` | object | Non-destructive display filters (section 5.2) | Observed |

Pixilart's UI offers these blend modes: Normal, Clipping Mask, Destination
Out, Destination Atop, Lighter, Multiply, Overlay, Darken, Color Dodge,
Color Burn, Difference, Saturation, Luminosity. **Reported.** The stored
strings are almost certainly the corresponding Canvas 2D operation names
(`source-over`, `destination-out`, `destination-atop`, `lighter`,
`multiply`, `overlay`, `darken`, `color-dodge`, `color-burn`, `difference`,
`saturation`, `luminosity`, and probably `source-atop` for Clipping Mask).
**Inferred.**

### 5.2 `options.filter`

Values mirror CSS `filter()` syntax and are applied at render time, not
baked into the layer PNG. Defaults shown.

| Field | Default | Notes | Tag |
|-------|---------|-------|-----|
| `brightness` | `"100%"` | | Observed |
| `contrast` | `"100%"` | | Observed |
| `grayscale` | `"0%"` | | Observed |
| `blur` | `0` | Pixels | Observed |
| `hue-rotate` | `0` | Degrees. UI range -180 to 180. Added around April 2024 | Observed, Reported |
| `dropshadow_x` | `0` | Pixels | Observed |
| `dropshadow_y` | `0` | Pixels | Observed |
| `dropshadow_blur` | `0` | Pixels | Observed |
| `dropshadow_alpha` | `1` | 0 to 1 | Observed |
| `dropshadow_color` | `"#000000"` | Note the leading `#`, unlike palette colours. Added around 2026 | Observed |

All filters in every sample were at their defaults, so the exact rendering
of non-default values is unverified.

---

## 6. Pixel data encoding

### The data URI

Every `src` and `preview` is:

```
data:image/pngp98kjasdnasd983/24kasdjasdbase64,<base64 PNG>
```

A standard PNG data URI is `data:image/png;base64,`. Pixilart replaces the
`;` with the fixed token `p98kjasdnasd983/24kasdjasd`. **Observed** in every
genuine file from version 1 through 2.7.0, and the same constant is used by
every converter and reader. The voidsprite importer's source comments that
the correct inverse is to replace the token with `;`. **Reported.**

Robust readers should simply split on the first `,` or search for
`base64,`, then base64-decode the remainder.

### The junk token in the top-level preview

The document-level `preview` (and only that field) has a second token
spliced **inside** the base64 payload:

```
iVBORw0KGgoAAAANSUhEUgAAAGQA/sfR5H8Fkddasdmnacvx/AABkCAYAAABw4pVU...
                            ^ offset 28
```

The token is `/sfR5H8Fkddasdmnacvx/` and it sits at base64 character
offset 28 in every 2.6 and 2.7.0 sample written by the web app (30 files).
Remove all occurrences before decoding. **Observed.** It is absent from all
71 version 1 samples and present in the version 2.1 converter template, so
it arrived with 2.x. The one 2.7.0 file with `previewApp: "pixil"` also
lacks it, another sign that file came from a different client.
The voidsprite source also mentions a third token,
`/8745jkhasdASD945kjknhj/`, tied to a flag named `importExportIsPhotoUpload`
in Pixilart's client code. It was not seen in any file. **Reported.**

### The PNG itself

- 8-bit RGBA, colour type 6, non-interlaced. Chunk layout is
  `IHDR`, one or two `IDAT`, `IEND`, which matches browser
  `canvas.toDataURL()` output. **Observed.**
- Dimensions always equal the canvas `width` x `height`. **Observed.**
- Layer PNGs hold raw pixels only. Opacity, blend and filters are not
  applied to them. The frame `preview` has them applied. **Observed** for
  opacity 1 and default filters.
- Pixilart's PNG export is the frame `preview` bytes, so a
  PNG to `.pixil` to PNG round-trip is lossless. **Observed.**

---

## 7. Compositing model

To render one frame, verified against stored previews:

1. Start with a transparent canvas of `width` x `height`.
2. For each layer in `layers` **in array order** (index 0 first, so it is at
   the bottom):
   - skip if `active` is `false`;
   - apply `options.filter` to the layer image;
   - draw it with `globalAlpha = opacity` and
     `globalCompositeOperation = options.blend`.

This is exactly the HTML Canvas 2D pipeline, which is what Pixilart runs on.
Third-party readers that composite in this order (paintcraft, voidsprite)
reproduce the previews. The khs2325 importer instead treats `active` as
unknown editor state and keeps every layer visible; that is a conservative
choice, not evidence about the format.

The document-level `preview` is a separately rendered thumbnail. In two
files it differed from the frame preview by a single pixel or by a faint
alpha on transparent pixels, so do not treat it as authoritative pixel data.
**Observed.**

---

## 8. Animation

- `frames` is the timeline in playback order. **Observed** on a genuine
  two-frame file.
- Each frame's `speed` is its duration in milliseconds. The two-frame
  sample used 250 ms per frame. GIF converters map GIF centisecond delays
  to `speed` by multiplying by 10. **Observed, Reported.**
- Every frame carries its own complete `layers` array with its own PNGs.
  There is no cel sharing or linking; identical content is stored twice.
- A logical layer that spans frames keeps the same `unqid` and `name` in
  each frame. The khs2325 importer requires this consistency. **Observed.**
- **Persistent Layers** is a Pixilart feature that keeps the layer list the
  same across all frames. Pixilart's help says it can be turned off once
  per drawing and never turned back on. `persLayers: true` at the top level
  is presumably its flag, but most files omit the field entirely.
  **Reported, Inferred.**
- The top-level `speed` appears to be the default for newly added frames.
  **Inferred.**
- Pixilart's help states there is no limit on frame count. **Reported.**

---

## 9. Type and value quirks

Pixilart's own writer is inconsistent, so readers must be lenient:

| Field | Variants seen |
|-------|---------------|
| `version` | `1` (int), `2.1`, `2.6` (float), `"2.7.0"` (string) |
| `width`, `height` | int in most files, numeric string (`"8"`) in others, including one saved in 2026 |
| `opacity` | numeric string `"1"` in 2.6 and 2.7.0, number `1` in version 1 |
| `currentFrame` | int, or string `"1"` in one version 1 file |
| `created_at` | one local file had `9998140154435`, a value in the year 2286, alongside a sane `updated_at`. Treat timestamps as untrusted |
| Missing fields | `alpha_lock`, `hue-rotate`, `dropshadow_color`, `persLayers`, frame `width`/`height` are all absent in some genuine 2.7.0 files |

Converters that emit `opacity: 1` as a number, integer dimensions, a tiny
custom palette and no `edit`, `alpha_lock`, `hue-rotate`, `dropshadow_color`
or timestamps report that Pixilart loads their files. **Reported.**

---

## 10. Version history

Dates come from `created_at` fields and repository history.

| Version | Seen | Distinguishing features |
|---------|------|-------------------------|
| `1` | 2017 to 2018 (71 files across six repos created in 2017) | No `type`. Palettes `default colors`, `common`, `skin tones`, `metro ui`, `rainbow dash`. Layer has only `id`, `src`, `name`, `opacity` (number), `active`, `unqid`. No `options`, `edit`, timestamps. No junk token in top preview |
| `2.1` | png_to_pixil template (circa 2019) | Same shape as v1 with `version: 2.1`. Junk token present in top preview |
| `2.6` | 2022 to early 2024 | Palettes renamed to `default`, `simple`, `common`, `skin tones`. Adds `colorSelected`, `palette_id`, layer `edit`, `options` with `blend`, `locked`, and an 8-key `filter`. `opacity` becomes a string |
| `2.7.0` | 2023 to present | Adds `type`, top-level `speed`, `previewApp`, `art_edit_id`, `created_at`, `updated_at`, `id`. Later additions without a version bump: `hue-rotate` (by April 2024), `alpha_lock` (by mid 2025), `dropshadow_color` (by January 2026), optional `persLayers`, `isExternal`, frame `data_id` |
| `2.8.0` | GitHub files from March 2026 | **Not genuine.** Written by a third-party script. Uses `application: "pixilart"`, a flat `colors` list, `hidden`, `undos`, `redos`. Do not treat as the format |

Because Pixilart adds fields without bumping `version`, readers should key
on field presence rather than version number.

---

## 11. Minimal valid file (writer template)

A single-frame, single-layer document in the current 2.7.0 shape. Replace
`<B64>` with the base64 of an RGBA PNG at exactly `width` x `height`, and
`<B64J>` with the same base64 but with `/sfR5H8Fkddasdmnacvx/` inserted
after character 28. Pixilart is known to accept files that omit the junk
token and most metadata, so the safest approach for a converter is to
mirror a real file as closely as practical.

```json
{
  "application": "pixil",
  "type": ".pixil",
  "version": "2.7.0",
  "website": "pixilart.com",
  "author": "https://www.pixilart.com",
  "contact": "support@pixilart.com",
  "width": 16,
  "height": 16,
  "colors": { "default": ["000000", "ffffff"] },
  "colorSelected": "default",
  "frames": [
    {
      "name": "",
      "speed": 100,
      "layers": [
        {
          "id": 0,
          "src": "data:image/pngp98kjasdnasd983/24kasdjasdbase64,<B64>",
          "edit": true,
          "name": "Background",
          "opacity": "1",
          "active": true,
          "unqid": "a1b2c3d4e5",
          "options": {
            "blend": "source-over",
            "alpha_lock": false,
            "locked": false,
            "filter": {
              "brightness": "100%",
              "contrast": "100%",
              "grayscale": "0%",
              "blur": 0,
              "hue-rotate": 0,
              "dropshadow_x": 0,
              "dropshadow_y": 0,
              "dropshadow_blur": 0,
              "dropshadow_alpha": 1,
              "dropshadow_color": "#000000"
            }
          }
        }
      ],
      "active": true,
      "selectedLayer": 0,
      "unqid": "f1g2h",
      "preview": "data:image/pngp98kjasdnasd983/24kasdjasdbase64,<B64>",
      "width": 16,
      "height": 16
    }
  ],
  "currentFrame": 0,
  "speed": 100,
  "name": "Untitled",
  "preview": "data:image/pngp98kjasdnasd983/24kasdjasdbase64,<B64J>",
  "art_edit_id": 0,
  "palette_id": false,
  "created_at": 1789180556055,
  "updated_at": 1789180556055,
  "id": 1789180556055
}
```

For an animation, emit one frame object per source frame, set each
`speed` in milliseconds, and give the layer the same `unqid` in every frame.
For multiple layers, append them bottom to top with `id` equal to their
index and set each frame's `preview` to the flattened composite.

---

## 12. Reading algorithm

1. Parse JSON. Reject if `frames` is missing or empty.
2. Read `width` and `height`, coercing numeric strings to integers.
3. For each frame, read `speed` (ms) and iterate `layers` in order.
4. For each layer `src`: take the substring after the first `,`, delete every
   `/sfR5H8Fkddasdmnacvx/`, base64-decode, and decode the PNG. Verify it is
   RGBA at canvas size.
5. Coerce `opacity` to a float. Treat `active: false` as hidden.
6. Composite per section 7 if a flattened image is needed, or use the
   frame `preview` directly when only the flattened result matters and
   fidelity to Pixilart's own render is preferred.

---

## 13. Open questions

- Exact stored strings for non-Normal blend modes and rendering of
  non-default filters. No sample used them.
- Meaning of frame-level `active`, `data_id`, top-level `isExternal`,
  `previewApp` and `edit`.
- Whether `persLayers` absence means the feature is on, off, or simply
  predates the field.
- Pixilart's maximum canvas size. One community converter refuses images
  over 1000 x 1000 but that is the converter's own limit.
- Whether Pixilart validates anything beyond `frames[].layers[].src` and
  the canvas size when opening a file.

---

## 14. Sources

Local samples: `resources/New One.pixil`, `resources/Test image AC.pixil`,
and the matching PNG exports in `resources/`.

Third-party code that reads or writes `.pixil`:

- [12joan/png_to_pixil](https://github.com/12joan/png_to_pixil), Ruby PNG and GIF to `.pixil` converter with a version 2.1 template.
- [spentine/image_to_pixil](https://github.com/spentine/image_to_pixil), Python and browser converter by Doduodrio and Spentine, handles GIF frames. Live at [spentine.github.io/image_to_pixil](https://spentine.github.io/image_to_pixil/web/).
- [counter185/voidsprite](https://github.com/counter185/voidsprite), C++ pixel editor whose `io_jsonformats.cpp` imports `.pixil` and documents the junk tokens.
- [cuebitt/paintcraft](https://github.com/cuebitt/paintcraft), TypeScript importer with typed interfaces and compositing tests.
- [khs2325/sprite-converter](https://github.com/khs2325/sprite-converter), browser converter with its own [pixil-format.md](https://github.com/khs2325/sprite-converter/blob/main/docs/pixil-format.md) describing a 2.7.0 subset.
- [EdVinyard/SpaceWar-X16](https://github.com/EdVinyard/SpaceWar-X16), Python script extracting layers from `.pixil` files.

Sample files surveyed on GitHub include those in Dabolus/portfolio-data,
kelvinink/crc-721, BradleyLamitie/Genesis, BDSinner/TSR_Repository,
EdVinyard/SpaceWar-X16, satanas/pandemia (two-frame animation),
MartinBinaghi/control-stock-system and BerserkSpence810/Industrial_Capitalist.

Descriptive references:

- [FileInfo: PIXIL file](https://fileinfo.com/extension/pixil)
- [Pixilart: Layer options and blend modes](https://www.pixilart.com/photo/layer-options-blend-modes-e40de89c1edfb9e)
- [Pixilart forum: converting a GIF to a PIXIL file](https://www.pixilart.com/forum/pixel-art/converting-a-gif-file-to-a-pixil-file-4142)
- [Pixilart forum: how do you make a layer appear on all frames](https://www.pixilart.com/forum/general/how-do-you-make-a-layer-appear-on-all-frames-11105)
- [Medium: Uploading any drawing to Pixilart](https://medium.com/@ivan.didyk.777/uploading-any-drawing-to-pixilart-45d7cc2845f2) (not retrievable during research, listed for completeness)
