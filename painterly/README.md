# Painterly Surface OSL Shader: User Guide

## Overview

`PainterlySurface` turns plain geometry with a flat color into a painted, concept-art look: warm lit areas, cool hue-shifted shadows, a saturated band along the light/shadow edge, brushy breakup, a cool sky rim, and a soft dark contour.

It does not use the scene's lights. It computes its own key light from `LightDirection` and outputs the result as an emission closure, so what you set in the parameters is what you see on screen. The file is `painterly_surface.osl`.

It is built for Blender Cycles, but the code is standard OSL and runs in other OSL renderers with small changes (see [Troubleshooting and limitations](#troubleshooting-and-limitations)).

## Setup in Blender Cycles

1. **Enable OSL.** Render Properties → Render Engine: Cycles → check **Open Shading Language**. Device: CPU, or GPU with OptiX (NVIDIA only). CUDA, HIP and Metal do not run OSL.
2. **Add the shader.** In the material's Shader Editor, add Script (Add → Script → Script). Set it to External and pick `painterly_surface.osl`, or Internal and paste the code into a Text block. Press the refresh icon to compile.
3. **Connect it.** Plug the Script node's `Out` socket into Material Output → Surface. Nothing else is needed.
4. **Set color management.** Render Properties → Color Management → View Transform: **Standard**. AgX and Filmic compress and desaturate the colors, which flattens the painted look.
5. **Aim the light.** Set `LightDirection` to the direction pointing *toward* the light, in world space. To follow a Sun lamp, add drivers on the X/Y/Z fields that read the Sun's local Z axis (its `matrix_world` column 2).
6. **Set the base color.** Type a color into `BaseColor`, or connect an Image Texture for painted detail. A painted texture gives the biggest quality jump.
7. **Render with few samples.** The shader is mostly emission, so 16–64 samples is usually clean. Turn denoising off if it smears the brush texture.

## How the shader builds the image

The color is built in layers, each painted over the last. When a parameter seems to do nothing, a later layer is often covering it.

1. **Brush strokes.** Two stroke fields are made in `StrokeSpace`: a main one along `StrokeDirection` and a weaker crossing one. Each gives a texture value and one random value per brush dab. Later layers use these.
2. **Light mask.** `N · L` plus `LightOffset`, broken up by the strokes (`EdgeBreakup`), then softened (`Softness`). A traced ray toward the light zeroes it in cast shadow. `Bands` can posterize it.
3. **Lit / shadow mix.** The mask blends between lit color (`BaseColor × LightColor`) and shadow color (`BaseColor × ShadowTint`, darkened by `ShadowDarkness`).
4. **Terminator band.** A warm, saturated stripe along the edge between lit and shadow.
5. **Sky fill.** Adds `SkyColor` to upward-facing shadow areas.
6. **Rim.** Paints `SkyColor` along the silhouette, broken by the strokes.
7. **Highlight.** Adds a shaped highlight where the surface faces the light, skipped in cast shadow.
8. **Jitter.** Shifts value and hue per dab and slowly across the surface.
9. **Contour.** Darkens toward the silhouette with `OutlineColor`. This is the last layer, so it covers the rim if both are strong.

## Parameter reference

Each table lists the default, a useful range, and what you see when you lower or raise the value. Colors are linear RGB, as Blender shows them in the color picker.

### Base color

| Parameter | Default | Useful range | Lower / darker | Higher / brighter |
| --- | --- | --- | --- | --- |
| `BaseColor` | 0.85, 0.58, 0.47 (skin) | any color or texture | Every layer darkens with it: shadows, terminator and sky fill are all multiplied by it | Brighter overall; very bright values can clip in the highlight |

### Key light

| Parameter | Default | Useful range | Lower | Higher |
| --- | --- | --- | --- | --- |
| `LightDirection` | 0.4, -0.5, 0.75 | any direction | — | Rotating it moves the whole light/shadow split; Z up lights from above |
| `LightColor` | 1.0, 0.96, 0.88 (warm white) | near-white tints | A cooler tint gives overcast or moonlit light | A warmer or orange tint gives sunset light; it also tints the highlight |
| `LightOffset` | 0.0 | -0.5 to 0.5 | Shadow grows; more of the form falls into shadow, a moodier read | Lit area grows; shadow shrinks to a thin sliver, a high-key read |
| `Softness` | 0.06 | 0.01 to 0.3 | Crisp, graphic edge between light and shadow, close to cel shading | Smooth, airbrushed gradient; above 0.2 it starts to read as plain 3D shading |
| `Bands` | 0 | 0, or 2 to 5 | 0 keeps the soft painted gradient | 2 gives hard two-tone cel shading; 3 to 5 add mid-tone steps like posterized paint |
| `CastShadows` | 1 | 0 or 1 | 0: no cast shadows, only form shadow; faster | 1: objects shade each other with a crisp shadow shape |
| `ShadowBias` | 0.01 | 0.001 to 0.05 | Too low: speckled "shadow acne" on lit faces | Too high: shadows detach from contact points and look like they float |

### Shadow painting

| Parameter | Default | Useful range | Lower | Higher |
| --- | --- | --- | --- | --- |
| `ShadowTint` | 0.62, 0.62, 0.95 (cool violet) | muted cool colors | Grey tint: plain, neutral shadows, closer to standard 3D | More saturated blue or violet: stronger temperature contrast, more painterly |
| `ShadowDarkness` | 0.45 | 0.2 to 0.75 | Light, airy shadows; low-contrast, illustrative | Deep, dramatic shadows; above 0.7 shadows lose color and go muddy |

### Warm terminator band

| Parameter | Default | Useful range | Lower | Higher |
| --- | --- | --- | --- | --- |
| `TerminatorColor` | 1.0, 0.45, 0.32 (red-orange) | warm, saturated colors | — | Red-orange reads as blood under skin; yellow-orange reads as sunlit fabric |
| `TerminatorWidth` | 0.15 | 0.05 to 0.35 | A thin, sharp warm line on the shadow edge | A wide, glowing warm zone; past 0.3 it tints most of the form |
| `TerminatorStrength` | 0.45 | 0 to 0.8 | 0 removes the band; skin looks flat and dead | Vivid, saturated edge; above 0.7 it looks like a colored outline |

### Sky fill and rim

| Parameter | Default | Useful range | Lower | Higher |
| --- | --- | --- | --- | --- |
| `UpAxis` | 0, 0, 1 | (0,0,1) Blender, (0,1,0) Y-up tools | — | Sets which way counts as "up" for the sky fill |
| `SkyColor` | 0.35, 0.62, 1.0 (sky blue) | match your background | — | Tints both the sky fill and the rim; match it to the backdrop to ground the object |
| `SkyFill` | 0.25 | 0 to 0.6 | 0: shadows stay one flat color | Upward-facing shadows pick up blue; more depth, but above 0.5 shadows read as blue |
| `RimWidth` | 0.3 | 0.1 to 0.6 | A thin rim hugging only the outer edge | A wide rim wrapping into the form |
| `RimStrength` | 0.35 | 0 to 0.8 | 0: no rim; the silhouette sits flat against the background | A strong cool edge that separates the object from the background |

### Painted highlight

| Parameter | Default | Useful range | Lower | Higher |
| --- | --- | --- | --- | --- |
| `SpecSize` | 0.06 | 0.01 to 0.2 | A small, sharp glint: wet or glossy (eyes, lips) | A large, broad patch: satin or matte sheen |
| `SpecStrength` | 0.35 | 0 to 1 | 0: fully matte | A bright painted highlight; above 0.8 it can clip to white |

### Brush strokes

| Parameter | Default | Useful range | Lower | Higher |
| --- | --- | --- | --- | --- |
| `StrokeSpace` | "object" | "object", "world" | — | "object": strokes stick to the mesh as it moves. "world": strokes stay put and the object slides through them |
| `StrokeDirection` | 1, 0.3, 0.2 | any direction | — | The direction strokes run. Point it along the main form (down a limb, across a cheek) |
| `StrokeScale` | 6.0 | 1 to 30 | Big, chunky dabs, like a large brush | Small, fine strokes; past about 20 they read as noise. Depends on object size |
| `StrokeStretch` | 4.0 | 1 to 10 | 1: round, blotchy dabs, like sponge or stipple | Long, streaky strokes, like a flat brush dragged across |
| `StrokeWarp` | 0.6 | 0 to 1.5 | 0: dab edges line up on a visible grid | Wobbly, organic dab shapes; above 1.2 strokes lose their direction |
| `EdgeBreakup` | 0.12 | 0 to 0.4 | 0: smooth, clean shadow edge | Ragged, dry-brush shadow edge; above 0.3 shadow flecks scatter into the lit side |
| `ValueJitter` | 0.06 | 0 to 0.2 | 0: flat, even fill | Visible patchwork of lighter and darker dabs |
| `HueJitter` | 0.02 | 0 to 0.08 | 0: one exact hue | Dabs drift in hue, like mixed paint; above 0.05 it looks multicolored |
| `LargeVariation` | 0.08 | 0 to 0.25 | 0: uniform color across the whole object | Broad, slow shifts in value and hue across the surface; breaks the CG flatness |

### Soft contour

| Parameter | Default | Useful range | Lower | Higher |
| --- | --- | --- | --- | --- |
| `OutlineWidth` | 0.22 | 0.05 to 0.5 | A thin line right at the silhouette | A wide, soft darkening wrapping into the form |
| `OutlineColor` | 0.16, 0.09, 0.12 (warm dark) | dark, warm colors | Near black: hard, inked comic look | A dark color tinted toward the base: soft, painted edge |
| `OutlineStrength` | 0.5 | 0 to 1 | 0: no contour | A solid line; at 1 it fully covers the rim |

## Outputs and compositing

| Output | Type | What it holds | Use it for |
| --- | --- | --- | --- |
| `Out` | closure | The final color as emission | Connect to Material Output → Surface |
| `Col` | color | The same final color as a plain color | Mixing with other shaders, or an AOV for comp |
| `LightMask` | float | 0 in shadow, 1 in light, after softness, cast shadow and bands | An AOV to grade light and shadow separately in comp |

To get `Col` or `LightMask` into the compositor, add an AOV Output node in the material, connect the socket, and add a matching AOV under View Layer Properties → Shader AOV.

The reference look also depends on the comp: a soft bloom (Glare node, Fog Glow), a slight blur or lens softening, and film grain. Those effects come after the shader and are not in it.

## Recipes

Start from the defaults and change only the listed values.

| Look | Changes from the defaults |
| --- | --- |
| Painted skin under blue sky (the reference image) | `TerminatorStrength` 0.55, `Softness` 0.04, `SkyFill` 0.3, `RimStrength` 0.4, `SkyColor` sampled from the background |
| Hard cel / anime | `Bands` 2, `Softness` 0.01, `EdgeBreakup` 0, `ValueJitter` 0, `HueJitter` 0, `TerminatorStrength` 0.2, `OutlineStrength` 0.9, `OutlineColor` 0.02, 0.02, 0.03 |
| Loose oil painting | `StrokeScale` 3, `StrokeStretch` 2.5, `StrokeWarp` 1.0, `EdgeBreakup` 0.25, `ValueJitter` 0.12, `HueJitter` 0.04, `LargeVariation` 0.15 |
| Soft watercolor | `Softness` 0.15, `ShadowDarkness` 0.25, `StrokeStretch` 1.2, `ValueJitter` 0.1, `OutlineStrength` 0.15, `SpecStrength` 0 |
| Sunset key light | `LightColor` 1.0, 0.7, 0.45, `TerminatorColor` 1.0, 0.3, 0.15, `ShadowTint` 0.5, 0.5, 0.9, `ShadowDarkness` 0.55 |
| Matte fabric | `SpecStrength` 0.05, `SpecSize` 0.15, `StrokeStretch` 6, `StrokeDirection` along the fold lines |

## Troubleshooting and limitations

| Symptom | Cause | Fix |
| --- | --- | --- |
| Object renders pink or black | OSL is off, or the device does not support it | Enable Open Shading Language; render on CPU or OptiX |
| Colors look washed out or grey | AgX or Filmic view transform | Set View Transform to Standard |
| Moving the scene's lamps changes nothing | The shader ignores scene lights by design | Change `LightDirection`, or drive it from a Sun lamp |
| Light comes from the wrong side | `LightDirection` points away from the light | Flip the vector's sign |
| Speckles on lit faces | Shadow ray hits its own surface | Raise `ShadowBias` toward 0.02–0.05 |
| Shadows don't touch the ground | `ShadowBias` too high | Lower `ShadowBias` |
| Strokes swim during animation | `StrokeSpace` is "world" | Set `StrokeSpace` to "object" |
| Strokes are huge on one object and tiny on another | Stroke size is in object units | Adjust `StrokeScale` per object, or apply scale |
| Visible grid pattern in the dabs | `StrokeWarp` too low | Raise `StrokeWarp` to 0.5 or more |
| No contour on flat surfaces | The contour comes from the facing angle, so flat faces have none | Add Freestyle lines or an inverted-hull outline |
| Rim light disappears | `OutlineStrength` covers it (contour is the last layer) | Lower `OutlineStrength` or `OutlineWidth` |

Limitations:

- **One key light.** Fill and bounce come only from `SkyFill` and the rim. More lights need more light parameters in the code.
- **The object doesn't light other objects.** The output is emission, so with high sample counts it can add faint glow to nearby surfaces. Set the material's Ray Visibility to camera only if that matters.
- **Contours are approximate.** Lines come from the facing angle, so they thicken on curved areas and vanish on flat ones. For ink-quality lines, combine with Freestyle or Grease Pencil Line Art.
- **Other renderers.** In Arnold, RenderMan or V-Ray, replace `emission()` with that renderer's emission or flat closure, and check that `trace()` is supported. Set `UpAxis` to `0, 1, 0` for Y-up applications.
