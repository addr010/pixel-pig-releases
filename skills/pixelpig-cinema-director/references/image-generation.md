# Image Generation

## Lock References Before Generating

Ask whether the character already exists. For an existing character, inspect supplied references and mirror the visible identity markers, face, skin, hair, build, expression, makeup, and wardrobe. For a new character, develop those details with the user. Wait for confirmation before the first generation.

Use visual descriptors rather than character names in model prompts. Describe only visible details from references. Avoid real brands, protected-IP names, age labels, and phrases such as "as before". Make every prompt stand alone.

## Build Characters In Order

Create one full-body character/outfit base before a character sheet. Capture garments, fabrics, colors, fit, layers, footwear, jewelry, accessories, props, hair/makeup changes, and visible markers. Generate one image on a pure white seamless studio background and wait for approval.

Prefer a connected GPT Image workflow after confirming the live schema:

- New base: `fal-text-to-image`, commonly `openai/gpt-image-2`.
- Reference-guided base: `fal-image-to-image`, commonly `openai/gpt-image-2/edit`, with character and wardrobe references in `files[]`.

Use a 16:9 2K output when available. Structure the base prompt as: locked visual character descriptor; head-to-toe wardrobe; controlled full-body pose; pure white seamless background; soft key, fill, and rim lighting; full footwear visible; photoreal finish.

Only after the base is approved, create one six-panel character sheet as a single horizontal 3x2 grid from that base. Use one prompt, one workflow run, one output, and thin white gutters. Default panels:

1. Full-body front.
2. Full-body three-quarter turn.
3. Full-body back.
4. Waist-up portrait.
5. Hands/accessory detail.
6. Face close-up.

Repeat the complete identity and wardrobe lock in the prompt and require identical identity, styling, proportions, lighting, and white background in every panel. Swap individual panel subjects only when requested; never turn the sheet into six independent generations.

## Create Scene Plates Only When Requested

Do not proactively generate a setting when the user asked only for character work. For a scene, plate, environment, or moment, choose either:

- A character-in-environment plate using locked references.
- A pure environment plate for later reference-to-video.

Use `fal-text-to-image` for a new plate or `fal-image-to-image` for reference-guided placement after describing the workflow. Include location, architecture, materials, time, weather, set dressing, props, atmosphere, palette, subject action, lighting direction and temperature, framing, lens language, focus plane, and grade.

## Use Detail Mode Deliberately

Use a high-quality image model for chest-up portraits, face/skin/eye/hair fidelity, key art, or exacting edits. Request the aspect ratio and quality supported by the described workflow. Include precise pose, crop, background, beauty lighting, real skin texture, iris moisture and pattern, lip detail, hairline/flyaways, fabric weave, and photographic finish.

## Apply A Consistent Photoreal Finish

Unless the user requests stylization, specify photographic skin texture, natural imperfections, strand-level hair and flyaways, realistic fabric weight and weave, reflective moist eyes, physically plausible metal, fine film grain, subtle edge aberration and vignette, and a cinematic grade. For environment-only plates, omit human-specific texture language.

Use one of these scene grammars when useful:

- Narrative: lived-in locations, natural handheld movement, vintage anamorphic feel.
- Studio/editorial: clean controlled set, tripod or slow push, saturated polished grade.
- Action: physical motion, debris, sustained handheld energy, gritty texture.
- Crowds: layered public activity, documentary immediacy, practical ambient light.
- Atmospheric/empty: negative space, locked-off or slow drift, palette-led weather and light.

## Use Adjacent Image Workflows When They Fit

- `fal-virtual-try-on`: preview a garment on a locked character; do not treat it as the canonical base until approved.
- `fal-image-reframe`: retarget a locked image to another aspect ratio.
- `fal-image-to-3d`: create a textured mesh and thumbnail for 3D use.
- Connected image-upscale workflows: upscale only approved hero assets, not every draft.

Always re-check the current workflow and model names with `pixelpig_list_workflows` and `pixelpig_describe_workflow`.
