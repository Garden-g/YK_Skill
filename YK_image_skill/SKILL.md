---
name: YK_image_skill
description: >
  Generate professional product images from user-uploaded photos using templates.
  Supports two main workflows: (1) Upload a product image → AI recognizes the product type
  and visual features → match and present suitable templates → user selects a template →
  generate or edit the image; (2) Upload a reference image → analyze it → save as a new
  reusable template. Use this skill whenever the user wants to generate product photos,
  create product marketing images, make product scene images, edit product images with
  templates, create product image templates, or mentions "产品图", "生图", "场景图",
  "模板", "product image", "product photo", "scene image". Also use when the user uploads
  an image and asks to generate variations, apply a style, place the product in a scene,
  or create marketing visuals.
---

# Product Image Generator

Generate professional product images by combining AI visual recognition, reusable prompt
templates, and image generation/editing tools. Speak Chinese.

## Available Tools

- **see_image** — Analyze images: identify product type, visual features, colors, materials
- **image_generate** — Generate a brand-new image from a text prompt
- **image_edit** — Edit an existing image based on text instructions (background swap, style transfer, etc.)

## Template System

Templates live in `templates/` within this skill's directory. Each template is a standalone
YAML file (`.yaml`) that the agent reads at runtime. Adding new templates is as simple as
dropping a new `.yaml` file — no code changes needed.

**Templates are exclusively user-created.** The skill does not ship with any preset templates.
All templates come from users uploading reference images through Workflow B. This ensures every
template has a real reference image and reflects the user's actual needs. If the templates
directory is empty when a user wants to generate an image, guide them to create a template first
or offer to generate with a custom prompt.

### Template File Format

Every template YAML file must follow this structure:

```yaml
name: "Template display name"
description: "One-line summary shown to the user during selection"
reference_image: "references/template-name.png"  # Path to the saved reference image (relative to templates/)
applicable_products:
  - "product_type_1"      # e.g. pillow, mug, lamp, chair, sofa
  - "product_type_2"
applicable_scenes:
  - "scene description 1"  # e.g. living room, studio, outdoor garden
  - "scene description 2"
style: "visual style keywords"  # e.g. minimalist, warm, Scandinavian, luxury
variables:
  - name: "variable_name"
    description: "What this variable controls"
    default: "fallback value if user doesn't specify"
prompt_template: |
  The actual prompt text with {{variable_name}} placeholders.
  Describe the scene, lighting, composition, and style.
  Use {{product_description}} for the product itself.
edit_prompt_template: |
  Optional. If present, used when user wants to edit the original image
  instead of generating a new one. Uses the same {{variables}}.
```

**Key fields:**
- `reference_image` — **critical**: path to the locally saved reference image for this template.
  When a template has a reference image, `image_edit` can use it as a base to swap in the
  user's product, producing results that faithfully match the original scene rather than
  being generated from scratch. Stored in `templates/references/`. This field is what makes
  template-based editing accurate — without it, the agent can only generate from text prompts.
- `applicable_products` — list of product types this template works well with (used for matching)
- `applicable_scenes` — helps the user understand where this template shines
- `variables` — customizable parts of the prompt; each has a name, description, and default
- `prompt_template` — the generation prompt with `{{variable}}` placeholders
- `edit_prompt_template` — (optional) alternative prompt for image_edit mode

## Workflow A: Generate / Edit Product Image

This is the primary flow. Follow these steps in order.

### Step 1: Recognize the Product

When the user provides a product image, call `see_image` to analyze it. Extract:

1. **Product type** — What category? (pillow, mug, lamp, chair, sofa, vase, candle, bag, etc.)
2. **Key visual features** — Color, material, texture, shape, pattern, notable design details
3. **Current scene context** — What background/environment is in the current image, if any

Summarize the recognition result to the user in one concise sentence, e.g.:
> "I see a navy blue velvet throw pillow with gold geometric embroidery."

### Step 2: Load and Match Templates

Read all `.yaml` files from the `templates/` directory within this skill folder.

**Matching logic:**
- Compare the recognized product type against each template's `applicable_products` list
- Matching is case-insensitive and supports partial matches (e.g., "throw pillow" matches "pillow")
- Collect all templates whose `applicable_products` contain the recognized product type

**If matches are found:** proceed to Step 3 with the matched templates.

**If no matches are found:** show ALL available templates to the user with a note:
> "I didn't find templates specifically designed for [product type], but here are all available templates — any of them can be adapted:"

**If the templates/ directory is empty or all files are invalid:** inform the user clearly:
> "There are no templates available yet. I can either (1) generate an image with a custom prompt based on your product, or (2) help you create a template from a reference image for future use. What would you prefer?"

### Step 3: Present Templates to the User

Display matched (or all) templates in a structured list. For each template show:

```
1. **[Template Name]** — [description]
   Applicable scenes: [scenes list]
```

Example:
```
1. **Cozy Living Room** — Warm home setting with natural light and soft textures
   Applicable scenes: living room, bedroom, reading nook

2. **Minimalist Studio** — Clean white studio with dramatic lighting
   Applicable scenes: product photography, e-commerce, catalog
```

Then ask the user to choose:
> "Which template would you like to use? You can also tell me any customization preferences (background color, lighting mood, specific scene details)."

### Step 4: Collect User Input and Build the Final Prompt

After the user selects a template:

1. Read the selected template's full YAML content
2. Ask the user if they want to customize any variables, showing the variable names and defaults:
   > "This template has these customizable options:
   > - **lighting**: warm golden hour (default)
   > - **background_detail**: wooden bookshelf with plants
   >
   > Want to change any of these, or shall I use the defaults?"
3. Fill in the `{{product_description}}` variable with the features recognized in Step 1
4. Replace all `{{variable}}` placeholders with user-specified or default values
5. The result is the final prompt

### Step 5: Generate or Edit

Determine the generation mode based on user intent and template capabilities:

#### Mode 1: Template has a reference image (recommended for faithful scene reproduction)

When the selected template has a `reference_image` field, this is the **preferred editing path**
because it produces results that faithfully match the original template scene.

- Call `image_edit` with **both** the template's reference image AND the user's product image
  as `reference_images`, plus the `edit_prompt_template` as the instruction
- The edit prompt should instruct: replace the product in the reference scene with the user's
  product, keeping the scene composition, lighting, and style intact
- This approach produces much more accurate results than generating from a text prompt alone,
  because the model can see the exact scene it needs to reproduce

**Example `image_edit` call with reference image:**
- `reference_images`: [template's reference image path, user's product image path]
- `prompt`: "Replace the product in this scene with {{product_description}}. Keep the exact
  same background, lighting, composition, and style. The product should look naturally placed
  in the scene."

#### Mode 2: Generate a new image (no reference image, or user explicitly wants new)

- Use `image_generate` with the final prompt from `prompt_template`
- This mode generates entirely from the text prompt — suitable when no reference image exists
  or the user wants a completely fresh creation

#### Mode 3: Edit the user's original photo

- Use `image_edit` with the user's original image and the `edit_prompt_template`
- Used when the user wants to modify their existing photo (change background, adjust style, etc.)

**Choosing the right mode:**
- Template has `reference_image` + user wants to use the template scene → **Mode 1** (reference-based edit)
- User says "generate a new photo", "create from scratch", or no reference image available → **Mode 2** (generate)
- User says "change the background", "edit my photo", "keep my product but change the setting" → **Mode 3** (edit original)
- If ambiguous and template has a reference image, **default to Mode 1** — it gives the best results
- If ambiguous and no reference image, ask the user

After generation, show the result to the user and offer:
> "Here's the result. Want me to adjust anything, try a different template, or generate another variation?"

---

## Workflow B: Create a New Template from a Reference Image

When the user wants to save a reference image as a reusable template, follow this flow.

**The reference image is the soul of the template.** Without it, future edits can only rely on
text prompts, which means the model has to "imagine" the scene from words alone — the result
will never faithfully match the original. With the saved reference image, `image_edit` can see
the exact scene and swap in a new product accurately. So saving the image locally is mandatory,
not optional.

### Step 0: Save the Reference Image Locally

Before any analysis, download and save the user's image to the `templates/references/` directory.

- Create the `templates/references/` folder if it doesn't exist
- Filename: use the template name (to be determined after analysis, so use a temp name first,
  then rename after the template name is finalized). Convention: `{template-name}.{ext}`
  (e.g., `rustic-wooden-stool-display.png`)
- Use `bash` with `curl` to download the image: `curl -sL -o <path> <url>`
- If the image is a local file path, use `bash` with `cp` to copy it
- **Verify the file was saved** by checking its existence and size
- This step is NON-NEGOTIABLE — do not skip it, do not defer it

### Step 1: Analyze the Reference Image

Call `see_image` on the user's image. Extract:

1. **Product type(s)** shown in the image
2. **Visual style** — minimalist, luxury, rustic, Scandinavian, industrial, etc.
3. **Scene / environment** — studio, outdoor, lifestyle, room setting, abstract background
4. **Lighting** — natural, warm, cool, dramatic, soft, hard
5. **Color palette** — dominant and accent colors
6. **Composition** — centered, angled, flat-lay, close-up, environmental
7. **Key elements** — props, textures, background objects that define the scene
8. **What varies** — what parts of this scene would change when used with different products

### Step 2: Generate Template Metadata

Based on the analysis, construct the template YAML structure:

- `name` — a descriptive, memorable name (e.g., "Scandinavian Morning Light")
- `description` — one sentence summarizing the template's vibe
- `reference_image` — **required**: relative path to the saved reference image
  (e.g., `references/scandinavian-morning-light.png`). Now rename the temp file from Step 0
  to match the finalized template name
- `applicable_products` — product types that fit well in this scene
- `applicable_scenes` — scene categories
- `style` — style keywords
- `variables` — identify 2-4 meaningful variables users might want to customize
  (lighting, color tone, background detail, mood, etc.)
- `prompt_template` — a detailed, well-crafted prompt that recreates the reference image's
  feel, with `{{variable}}` placeholders for customizable parts
- `edit_prompt_template` — a corresponding edit instruction. When a reference image exists,
  this prompt should focus on "replace the product in this scene" rather than "recreate this
  scene from scratch", because the reference image provides the visual context

### Step 3: Confirm with the User

Present the generated template metadata to the user in a readable format:

> **Template Preview:**
> - Name: Scandinavian Morning Light
> - Description: Soft morning light on a light wood surface with linen textures
> - Suitable products: mug, candle, vase, small decor items
> - Scenes: kitchen, dining table, morning routine
> - Customizable: lighting (warm/cool), surface material, accent color
>
> Does this look right? Any changes before I save it?

### Step 4: Save the Template

After user confirmation:

1. **Rename the reference image** (if still using a temp name) to match the template name:
   `templates/references/{template-name}.{ext}`
2. **Write the YAML file** to the `templates/` directory:
   - Filename: derive from the template name, lowercased, hyphens for spaces
     (e.g., `scandinavian-morning-light.yaml`)
   - The `reference_image` field must point to the correct relative path
3. **Verify both files exist**: the `.yaml` template and the reference image in `references/`
4. Confirm to the user:
   > "Template saved! Reference image and metadata are both stored locally. When you use this
   > template in the future, the system will use the saved reference image to accurately
   > reproduce the scene with your new product."

---

## Edge Cases and Fallbacks

Handle these situations gracefully — never crash or show raw errors.

| Situation | Response |
|---|---|
| Templates directory is empty | Offer to generate with a custom prompt or create a template |
| No templates match the product | Show all templates with explanation |
| Template YAML is malformed | Skip the bad file, log which file had issues, continue with valid templates |
| User provides no image | Ask for an image to get started |
| Product type is ambiguous | State your best guess, show matching + a few extra templates, let the user correct |
| User wants both edit and generate | Do both sequentially — edit first, then generate, or ask for preference |
| Variable has no default and user doesn't specify | Use a sensible placeholder based on the product and scene context |
| `edit_prompt_template` missing when user wants edit | Use `prompt_template` as the edit instruction instead |

## Important Notes

- Always use `see_image` as the first step when the user provides an image — never skip recognition
- **When creating a template, always save the reference image locally first** — this is the foundation for faithful scene reproduction. Never skip this step, never defer it
- When a template has a `reference_image`, always prefer Mode 1 (reference-based edit) for the best results — the model can see the exact scene and swap in the product accurately
- Template matching is fuzzy and inclusive: when in doubt, include the template
- Keep template prompts detailed and specific — vague prompts produce poor images
- When building the final prompt, always mention the product type, material, and color explicitly
- The `{{product_description}}` variable is special: it's always auto-filled from see_image recognition, the user doesn't need to provide it

## Directory Structure

```
templates/
├── references/              ← Saved reference images (one per template)
│   ├── cozy-living-room.png
│   └── rustic-wooden-stool-display.png
├── cozy-living-room.yaml    ← Template YAML files
├── minimalist-studio.yaml
└── ...
```

The `references/` subfolder holds all saved reference images. Each image is linked from its
template's `reference_image` field. This separation keeps templates organized: YAML metadata
in `templates/`, image assets in `templates/references/`.
