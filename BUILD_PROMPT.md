# HouseDesign build prompt

**How to use this (notes for you, not part of the prompt)**

1. Fill in what you know under *About our house* below. It's all optional. The prompt assumes you're in the UK, so change that if it's wrong.
2. Start a new Claude Code session on this repository and say: **"Read BUILD_PROMPT.md and follow it, starting with M0."** In later sessions, say: **"Continue BUILD_PROMPT.md from docs/STATUS.md."**
3. This repository is **public**, so don't upload your floorplan or photos to GitHub. You add them inside the finished app, where they stay in your browser. If you share them with Claude during a session, the prompt tells it to keep them out of git. You could make the repository private instead, but the free GitHub Pages hosting used here needs a public repository.
4. The in-app AI features need an Anthropic API key. It's pay-as-you-go from the Claude Console and separate from a Claude subscription. Without a key, everything else still works, and Claude Code can do the AI steps for you.

---

<!-- Prompt starts here -->

# Build HouseDesign: a photorealistic 3D model of our new house, with an interior design app built around it

## The goal

We've bought a house but haven't moved in yet. Build a web app where I upload the floorplan and photos of the house, and Claude works out every measurement and builds an accurate, photorealistic 3D model from them. I'll then use the model as an interior design tool to plan our dream home before we move in: moving, adding and removing furniture, changing paint, flooring and lighting, and trying out different designs.

I also want to upload a photo of anything (a sofa from a shop's website, a lamp we already own, a print for the wall) and have Claude turn it into a 3D object of the right size that I can place and move around the house.

## About our house (fill in what you know; all optional)

- Country, units and spelling: UK. British English. Metric first (m and cm), with feet and inches alongside.
- Type and age: [e.g. 1930s three-bed semi / 2015 new build]
- Floors: [e.g. ground, first, loft conversion]
- What we have: [estate agent floorplan as PDF/JPG, about N listing photos, our own photos and videos from viewings]
- Measurements we know for certain: [e.g. ceiling heights, anything from a survey]
- What's staying with the house: [e.g. kitchen, bathroom suites, light fittings, curtains. In England and Wales this is listed on the TA10 fittings and contents form]
- Devices we'll use: [laptop, iPad, phone]
- Anthropic API key for the in-app AI: [yes / not yet]

## Principles

1. **Measured, not made up.** Every dimension in the model records three things: its value, its source (floorplan label, scaled from the floorplan, calibrated photo, photo estimate, standard default, or entered by me) and a confidence. Low-confidence values are flagged where I'll see them and are never presented as fact. Values I enter always win, and a later AI run never overwrites them.
2. **I stay in control.** AI results arrive as proposals that I can review, edit, accept or reject. They are never applied silently, and everything can be undone.
3. **Private by default.** This repository is public. Never commit or bundle real house data: floorplans, photos, generated house models or API keys. Images leave the device only when I start an AI action, and only to Anthropic's API (or to an image-to-3D service I've explicitly turned on). Any real files I share during development go under `private/`, which is gitignored. Use a fictional demo house for development, tests and screenshots.
4. **Works without AI.** Viewing, editing and designing need no API key. The AI features switch on once a key is set.
5. **Easy for a non-expert.** I'm not a CAD user. Use plain-English labels and sensible defaults, plus a guided first run: floorplan, check the rooms, photos, review the flagged measurements, done.

## Architecture (decided; tell me if you think any of it is wrong)

- A static single-page app built with Vite, React and TypeScript, using three.js through React Three Fiber and drei. Deploy it to GitHub Pages with a GitHub Actions workflow, and tell me which setting I need to switch on.
- One versioned project document (JSON, millimetres internally), validated with Zod. It lives in a Zustand store with undo and redo and autosaves to IndexedDB. I can export and import the whole project (data, images and models) as a single file, to back it up or move it between devices.
- Claude: use the official `@anthropic-ai/sdk`, called from the browser (`dangerouslyAllowBrowser: true`) with my own key. I enter the key in Settings, and it is stored only on this device.
  - Load the `claude-api` skill before writing any Claude API code, and follow it.
  - Default to the model the skill recommends, and let me change it in Settings.
  - Set the effort level per task: higher for reading the floorplan, lower for quick item specs.
  - Use structured outputs or strict tools, so every AI result is validated against the schema before it touches the project.
  - Mind the image limits: large images are downscaled, and stricter limits apply above 20 images in one request. Use the Files API for images that are reused across turns.
  - Show an estimated cost before any batch job, such as analysing every photo.
- Every AI step produces a JSON proposal in a documented format. The app can import proposals from a file as well as generate them through the API, so without an API key I can give my images to Claude Code and it writes the same proposal files for me to import. Document this in `docs/claude-code-workflow.md`.

## Data model (outline; refine as needed)

- **Levels** (floors). Each level has:
  - **walls**: centreline, thickness, height, external or internal
  - **openings** (doors, windows, archways, patio and bifold doors): host wall, offset, width, height, sill height, hinge side and swing
  - **rooms**: outline, name, floor, wall and ceiling finishes, ceiling height
  - **stairs** and **fixtures**
- **Items**: anything placeable. Each item has an asset reference, position, rotation, size, material overrides, room, layer, locked and hidden flags, notes, price and link.
- Objects come in three tiers:
  - *Structure* (walls, floors, ceilings, stairs, doors, windows): changed only in Renovate mode.
  - *Fixtures* (kitchen units and appliances, bathroom suite, radiators, fireplaces, built-in wardrobes, light fittings, sockets and switches): locked by default. They can be unlocked, moved, replaced or removed.
  - *Loose items* (furniture, lamps, rugs, curtains, art, plants, accessories): free to move and remove.
- **Sources**: floorplan pages, photos and product images. Each photo stores its camera calibration.
- **Measurement provenance** for every dimension (see principle 1).
- **Scenarios**: all scenarios share the structure. Each has its own items, finishes and lighting, so "Plan A" and "Plan B" can exist side by side.

## Working out the measurements

This is the hardest and most important part. Getting it right matters more than any other feature.

### From the floorplan

- Accept PDF, PNG, JPG, WebP and iPhone HEIC files (convert HEIC in the browser). Render PDF pages at high resolution with pdf.js, and also read the text layer with positions. Many estate agent PDFs are vector files, so their dimension labels can be read exactly.
- UK estate agent plans are usually marked "not to scale". Treat the labelled dimensions as the truth and the drawing as a guide to the layout.
  - Handle label forms such as `4.27m x 3.66m (14'0" x 12'0")`, `max`, `into bay`, `narrowing to` and `plus recess`.
  - Parse every unit format: m, cm, mm, `12'6"` and `12ft 6in`.
  - Work out which dimension in each label belongs to which drawn axis.
  - Use any "total approx. floor area" figure as a cross-check.
- Large plans are downscaled when sent to Claude, which can make small labels unreadable. Send the whole page for the layout plus higher-resolution crops, and give Claude a `crop_image` tool so it can zoom in wherever it needs to.
- Extract:
  - rooms and their dimensions
  - walls, and doors with their swing
  - windows, openings and bays
  - chimney breasts and alcoves
  - stairs: direction and number of steps
  - built-ins
  - anything ambiguous
- Reconcile everything into one consistent geometry with a least-squares solve:
  - Labelled internal dimensions are strong constraints, and drawn positions are weak priors.
  - Walls snap to right angles unless they are clearly angled (as in bays).
  - Use typical UK wall thicknesses as priors: external walls are about 230 mm for solid brick and 300 mm or more for cavity walls; internal walls are about 100–125 mm.
  - Rooms must fit together with no gaps or overlaps.
  - Report any label the solve couldn't satisfy.
- Stack the floors using the stairs and the external walls.
- Check the result. Render the reconstructed plan from above, show it to Claude next to the original and have it list the discrepancies. Fix them and repeat, up to a set number of rounds.

### From the photos

- Accept photos and videos. For videos, pull out sharp frames spread across the clip and treat them as photos.
- For each photo, work out:
  - which room it shows
  - roughly where the camera stood and which way it faced
  - what's visible: doors, windows, radiators, sockets, fixtures, finishes (floor type, wall colour, tiles, worktops) and loose furniture
- Some heights can't come from the floorplan: ceilings, door heads, window sills and heads, worktops, radiators, skirting, picture rails and coving. Work these out in two steps.
  1. Estimate them from objects of known size. UK internal doors are 1981 mm tall (often 762 mm wide), socket and switch plates are 86 mm tall, worktops are about 900 mm high and 600 mm deep, and brick courses are 75 mm.
  2. Refine them with **photo calibration**. Match points in the photo to points in the model (Claude proposes the matches and I can drag to correct them), solve the camera pose and focal length, and measure by back-projection. In a calibrated photo, items and fixtures marked in the image can be projected onto the floor to get their true position and size. Combine the estimates from every photo; how far they spread sets the confidence.
- **Photo-match view**: put the 3D camera at a photo's calibrated pose and overlay the original photo, with an opacity slider. This is how I (and you) check the model is right.
- Finishes: choose the closest material in the library and tint it to the colour sampled from the photo. White-balance against surfaces that are probably white, such as ceilings and woodwork.
- Rebuild the seller's furniture from the photos as approximate, movable items on an "As listed" layer, so that photo matching looks right. One click ("Empty the house") removes it all, since it won't be there when we move in.
- Produce a "Measure these on your next visit" checklist, ordered by how much each measurement would improve the model (ceiling heights, alcove widths, window sizes for blinds, and so on).

### Accuracy targets

Room dimensions should be within 5 cm of the labels (or of the labels' own rounding). Heights from calibrated photos should be within about 5 cm. Flag anything worse.

## The 3D model and realism

- Model the real architecture: walls with true openings, window reveals and sills, skirting, architraves, coving, doors that open, glazed windows, stairs with balustrades, chimney breasts and fireplaces, radiators, sockets and switches, and ceiling lights. Small details are what make it look real.
- Two quality modes:
  - **Interactive**, which must stay smooth while I edit: PBR materials, image-based lighting, soft shadows, ambient occlusion and physically based lights (lumens and colour temperature). Use Khronos PBR Neutral tone mapping so paint and fabric colours stay true.
  - **Photo mode**: progressive path-traced stills (for example with three-gpu-pathtracer) from any camera, saved to a render gallery and downloadable.
- Daylight: position the sun from the location, date and time (for example with suncalc) and the house's north bearing, taken from the plan or asked of me. This lets me see how light falls in each room through the day and across the seasons.
- Give the windows a believable view (sky, garden, neighbouring houses as simple shapes) rather than a void.
- Materials: a curated library of CC0 textures from Poly Haven and ambientCG, covering wood floors (including herringbone), carpets, tiles, stone, worktops, paint finishes, fabrics (linen, velvet, bouclé, leather), metals and glass. Compress the textures and record their licences.
- Performance: a smooth 60 fps while editing on a recent laptop or iPad, with quality presets for slower devices.

## The interior design app

- **Views**:
  - a 3D dollhouse view, with the walls nearest the camera cut away and the ceilings hidden
  - a 2D plan with dimensions
  - a first-person walkthrough with adjustable eye height, no walking through walls, and touch controls on iPad
  - the photo-match view
- **Catalogue**: parametric furniture with good proportions and rounded edges, not plain boxes. It covers:
  - sofas and armchairs
  - beds in UK sizes (single, small double, double, king, super king)
  - wardrobes and chests of drawers
  - tables, chairs and desks
  - TV units and shelving
  - lamps, rugs, curtains and blinds
  - mirrors, art and plants

  Every item can be resized and has editable materials.
- **Editing**: drag items along the floor with snapping (to a grid, walls and other items), rotate them, raise wall-mounted items, and type exact sizes and positions. Duplicate, delete, hide, lock and group items. Undo and redo are unlimited, and everything works by mouse, keyboard or touch.
- **Surfaces**: wallpaper, tiles, flooring, ceilings, woodwork, and paint per wall or per room. Paint can be set from a colour picker, a hex value, "match this photo", or a named paint colour that Claude approximates (with a note that screens aren't exact).
- **Lighting**: lamps and ceiling lights are real light sources. I can switch them on and off, dim them, and make them warmer or cooler.
- **Renovate mode**: move or remove internal walls, add or resize openings, and re-plan the kitchen and bathrooms. Show a reminder that structural changes need professional advice.
- **Helpful checks**:
  - distances to the walls while dragging
  - clearance warnings for walkways, around beds and in front of wardrobes
  - door-swing clashes
  - "Will it fit?": checks a large item against the doors, hallway and stairs it must pass through to reach its room
- **Quantities**, exportable:
  - floor areas for flooring and carpet
  - wall areas net of openings, with paint litres for two coats
  - skirting lengths
  - window sizes for blinds and curtains
- **Scenarios**: create, duplicate, rename, switch between and compare side by side.
- **Shopping list**: items with price, retailer and link, totals per room and overall, and CSV export.
- **Exports**: the project backup file, screenshots and renders, and a furnished 2D plan as PNG or PDF.

## Add anything from a photo

- I upload, paste or drag in a photo or screenshot. I can add a product link, the price, or the dimensions from the listing. If there's a link, Claude can use its web fetch tool to read the listed dimensions.
- Claude identifies the object and returns an item spec:
  - category and style
  - width, depth and height in millimetres, with a confidence and how it got them (read from the listing or estimated)
  - materials and colours
  - how it's mounted: floor, wall, ceiling, tabletop or window
  - parameters for the matching parametric model
- I see a 3D preview next to the photo and correct the size or colours if needed. The item then goes into "My items", ready to place anywhere, as often as I like.
- There are three ways to build the object:
  1. **Parametric** (the default; instant and free): the matching catalogue model, sized and coloured to match.
  2. **Flat items** (art, photos, posters, mirrors, rugs, wallpaper, tiles, flooring): use the image itself, framed, as a rug, or tiled across a surface. Remove the background in the browser where it helps.
  3. **Detailed** (optional): a pluggable image-to-3D service, with the mesh scaled to the confirmed real-world size. Find out which services are current and what they cost, and ask me before using anything paid.
- I can upload several product photos at once.
- I can add new photos of the house at any time (from our next visit, say). Claude matches them to rooms and proposes corrections to the model.

## Design assistant

- A chat panel for requests like "suggest a living room layout with our sofa and a reading corner" or "show this bedroom in a Japandi style", or for an inspiration photo with "make it feel like this".
- Claude works on the scene through tools. It can:
  - read a room
  - search the catalogue and My items
  - add, move and remove items
  - set finishes and lights
  - check clearances
  - render a view, so it can look at its own result

  It produces a change set that I preview and then accept or reject, usually as a new scenario.
- Validate every tool call against the schema before applying it.

## Milestones

Build in this order. Each milestone ends with the tests passing, the app deployed, `docs/STATUS.md` updated, and a short note telling me what to try.

- **M0, foundations**: the scaffold, CI, deployment, the schema and the demo house. Includes a 3D viewer with orbit and plan views, saving, loading and exporting projects, and a `CLAUDE.md` describing the architecture and commands.
- **M1, floorplan to model**: upload, extraction with the crop tool, the reconciliation solve and the measurement review panel. Walls, openings, floors, ceilings and stairs appear in 3D and 2D. **Pause here** so I can try our real floorplan.
- **M2, photos**: room matching, calibration, the photo-match overlay, heights, finishes and fixtures, plus the "As listed" layer and the next-visit checklist. **Pause here too.**
- **M3, design editor**: the catalogue, editing, surfaces, lighting, Renovate mode, scenarios, quantities and checks.
- **M4, add from photo**: item specs, the parametric, flat and optional detailed models, My items and the shopping list.
- **M5, realism**: quality presets, photo mode, daylight through the day and across the seasons, and the view out of the windows.
- **M6, design assistant.**

Later, if there's time:

- the garden and exterior
- sharing designs with the rest of the household
- importing a LiDAR room scan (for example from an iPhone Pro) as a high-accuracy measurement source
- a VR walkthrough (WebXR)

## Quality bar

- Unit tests for the geometry, unit parsing, the measurement solve and schema migrations.
- Recorded fixtures for the AI calls, with no live API calls in CI.
- Playwright end-to-end tests for the core flows.
- Test the pipelines against known answers:
  - Generate an estate-agent-style floorplan (as an image and as a PDF) of the demo house.
  - Render synthetic "listing photos" of it from known cameras.
  - Measure how close the extraction and calibration get to the true values.
- Look at your own work. Take screenshots with Playwright and inspect them, especially renders compared against the source photos.
- By the end, this whole story must work:
  1. On my iPad, I upload our floorplan and photos.
  2. I review a short list of flagged measurements and see our house in 3D.
  3. In photo-match, the living room lines up with the estate agent's photo.
  4. I empty out the seller's furniture and add our own sofa from a photo.
  5. I try two paint colours as two scenarios.
  6. I walk through in first person and render the living room in photo mode.
  7. I check the shopping list total.

## Working agreement

- Work autonomously within these decisions. Ask me only when you're blocked on something that's genuinely my call, such as paying for a service.
- Depth over breadth: finish each milestone properly before starting the next.
- Commit in small, clear steps. Keep `docs/STATUS.md` current (done, next, known issues) so a fresh session can pick up where you left off.
- Never commit `private/`, real photos or floorplans, generated house data or keys.
- Be honest about accuracy. When something is estimated, say so, both in the app and to me.
