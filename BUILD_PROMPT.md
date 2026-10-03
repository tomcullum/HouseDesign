# HouseDesign build prompt

**How to use this (notes for you, not part of the prompt)**

Everything runs on your laptop in the Claude desktop app: no cloud session, no website hosting and no API key. Claude's work counts towards your normal Claude plan.

1. In the Claude desktop app, open **Code** and choose **Local**. Click **No folder** and pick an empty folder for the project; for example, create one called `HouseDesign` in your Documents folder.
2. Send this as your first message: **"Download https://github.com/tomcullum/HouseDesign (branch `claude/festive-cannon-9ls7h4`) into this folder, then read BUILD_PROMPT.md and follow it, starting with M0."** Claude creates all the other folders itself.
3. In later sessions, open the same folder and say: **"Continue BUILD_PROMPT.md from docs/STATUS.md."**
4. When Claude asks, put your floorplan and photos into the `private/inputs` folder it creates. They stay on your laptop and are never uploaded to GitHub. Claude itself runs on Anthropic's servers, as every Claude session does, so the images it looks at are sent to it, but nothing else leaves your laptop.
5. Claude will ask about the house at the start. You can also fill in *About our house* below. The prompt assumes you're in the UK, so change that if it's wrong.

---

<!-- Prompt starts here -->

# Build HouseDesign: a photorealistic 3D model of our new house, with an interior design app built around it

## The goal

We've bought a house but haven't moved in yet. I want an app where I add the floorplan and photos of the house, and Claude works out every measurement and builds an accurate, photorealistic 3D model from them. I'll then use the model as an interior design tool to plan our dream home before we move in: moving, adding and removing furniture, changing paint, flooring and lighting, and trying out different designs.

I also want to give Claude a photo of anything (a sofa from a shop's website, a lamp we already own, a print for the wall) and have it turned into a 3D object of the right size that I can place and move around the house.

Everything runs locally on my laptop. You (Claude Code, in a local session like this one) are the AI: you read the floorplan and photos, work out the measurements, and turn photos into 3D items. The app is where I see, check and design the house.

## About our house (if any of this is blank, ask me briefly at the start)

- Country, units and spelling: UK. British English. Metric first (m and cm), with feet and inches alongside.
- Type and age: [e.g. 1930s three-bed semi / 2015 new build]
- Floors: [e.g. ground, first, loft conversion]
- What we have: [estate agent floorplan as PDF/JPG, about N listing photos, our own photos and videos from viewings]
- Measurements we know for certain: [e.g. ceiling heights, anything from a survey]
- What's staying with the house: [e.g. kitchen, bathroom suites, light fittings, curtains. In England and Wales this is listed on the TA10 fittings and contents form]
- Laptop: [Windows / Mac]

## How it all fits together (decided; tell me if you think any of it is wrong)

- **Local only.** The app runs on my laptop with one command, and as a preview inside the Claude desktop app. Don't add any of the following without asking me first: hosting, cloud services, an Anthropic API key, or anything else that costs money. One-off free downloads, such as CC0 textures or npm packages, are fine.
- **You are the AI.** All the AI work happens in Claude Code sessions like this one, on my normal Claude plan. The app itself never calls an AI service.
- **Files on disk are the source of truth.** They live in a gitignored `private/` folder:
  - `private/inputs/`: floorplans and photos of the house that I drop in
  - `private/inbox/`: photos of items I want to add
  - `private/project/`: the house model, scenarios, My items and renders. The app reads and writes these.
  - `private/proposals/`: your output. You read `private/project/` but never edit it directly.
- **Proposals.** Every AI step produces a JSON proposal file in a documented, schema-validated format. That includes reading the floorplan, analysing photos, adding an item and suggesting a design. The app notices new proposals straight away and shows them so I can review, edit, accept or reject them. Validate each proposal with a script (for example `npm run validate`) before telling me it's ready.
- **Slash commands for me**, as project skills in `.claude/skills/`:
  - `/scan-house`: build or update the model from `private/inputs/`
  - `/add-item`: turn new photos in `private/inbox/` into items
  - `/design`: suggest a layout or style as a new scenario

  If I attach an image in the chat and you need the file itself (for a texture, say), ask me to save it to `private/inbox/`.
- **Tools for you**: command-line scripts that let you check your own work by looking at it. They:
  - crop and enlarge images
  - read the text layer of PDFs
  - run the measurement solve
  - validate proposals
  - render any scenario from any camera to an image
- **Tech**:
  - Vite, React and TypeScript, with three.js through React Three Fiber and drei.
  - A small local Node server (or Vite middleware) that reads and writes `private/` and watches for new proposals.
  - One versioned project document (JSON, millimetres internally), validated with Zod and held in a Zustand store with undo and redo.
  - Export to a single backup file, and import from one, so I can copy the project to another computer.

## Principles

1. **Measured, not made up.** Every dimension in the model records three things: its value, its source (floorplan label, scaled from the floorplan, calibrated photo, photo estimate, standard default, or entered by me) and a confidence. Low-confidence values are flagged where I'll see them and are never presented as fact. Values I enter always win, and a later scan never overwrites them.
2. **I stay in control.** AI results arrive as proposals that I review in the app. They are never applied silently, and everything can be undone.
3. **Private and local.** This repository is public on GitHub. Never commit `private/` or any real house data: floorplans, photos or generated house models. Commit the code locally as you go, but don't push to GitHub unless I ask. Use a fictional demo house for development, tests and screenshots.
4. **The app works on its own.** Viewing, editing and designing never need a Claude session running.
5. **Easy for a non-expert.** I'm not a CAD user. Use plain-English labels and sensible defaults. The first run should guide me: put the files in `private/inputs/`, type `/scan-house` in Claude, review the result.

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

- Accept PDF, PNG, JPG, WebP and iPhone HEIC files (convert HEIC with a script). Render PDF pages at high resolution, and also read the text layer with positions. Many estate agent PDFs are vector files, so their dimension labels can be read exactly.
- UK estate agent plans are usually marked "not to scale". Treat the labelled dimensions as the truth and the drawing as a guide to the layout.
  - Handle label forms such as `4.27m x 3.66m (14'0" x 12'0")`, `max`, `into bay`, `narrowing to` and `plus recess`.
  - Parse every unit format: m, cm, mm, `12'6"` and `12ft 6in`.
  - Work out which dimension in each label belongs to which drawn axis.
  - Use any "total approx. floor area" figure as a cross-check.
- A large plan gets downscaled when you look at it, which can make small labels unreadable. Look at the whole page for the layout, then crop and enlarge regions with a script to read the labels.
- Extract:
  - rooms and their dimensions
  - walls, and doors with their swing
  - windows, openings and bays
  - chimney breasts and alcoves
  - stairs: direction and number of steps
  - built-ins
  - anything ambiguous
- Reconcile everything into one consistent geometry with a least-squares solve. Write it as unit-tested project code that runs from a script, rather than working it out in your head.
  - Labelled internal dimensions are strong constraints, and drawn positions are weak priors.
  - Walls snap to right angles unless they are clearly angled (as in bays).
  - Use typical UK wall thicknesses as priors: external walls are about 230 mm for solid brick and 300 mm or more for cavity walls; internal walls are about 100–125 mm.
  - Rooms must fit together with no gaps or overlaps.
  - Report any label the solve couldn't satisfy.
- Stack the floors using the stairs and the external walls.
- Check the result. Render the reconstructed plan from above, compare it with the original and list the discrepancies. Fix them and repeat, up to a set number of rounds.

### From the photos

- Accept photos and videos. For videos, pull out sharp frames spread across the clip and treat them as photos.
- For each photo, work out:
  - which room it shows
  - roughly where the camera stood and which way it faced
  - what's visible: doors, windows, radiators, sockets, fixtures, finishes (floor type, wall colour, tiles, worktops) and loose furniture
- Some heights can't come from the floorplan: ceilings, door heads, window sills and heads, worktops, radiators, skirting, picture rails and coving. Work these out in two steps.
  1. Estimate them from objects of known size. UK internal doors are 1981 mm tall (often 762 mm wide), socket and switch plates are 86 mm tall, worktops are about 900 mm high and 600 mm deep, and brick courses are 75 mm.
  2. Refine them with **photo calibration**. You propose matching points between the photo and the model; I can drag them in the app to correct them. The app then solves the camera pose and focal length and measures by back-projection. In a calibrated photo, items and fixtures marked in the image can be projected onto the floor to get their true position and size. Combine the estimates from every photo; how far they spread sets the confidence.
- When you give pixel coordinates, work on copies you've resized yourself to about 1800 px on the long edge or less. Then they aren't downscaled again before you see them, and your coordinates map back exactly.
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
  - **Photo mode**: progressive path-traced stills (for example with three-gpu-pathtracer) from any camera, saved to a render gallery.
- Daylight: position the sun from the location, date and time (for example with suncalc) and the house's north bearing, taken from the plan or asked of me. This lets me see how light falls in each room through the day and across the seasons.
- Give the windows a believable view (sky, garden, neighbouring houses as simple shapes) rather than a void.
- Materials: a curated library of free CC0 textures from Poly Haven and ambientCG, downloaded once into the project. Cover wood floors (including herringbone), carpets, tiles, stone, worktops, paint finishes, fabrics (linen, velvet, bouclé, leather), metals and glass. Compress the textures and record their licences.
- Performance: smooth while editing on my laptop, with quality presets if it struggles.

## The interior design app

- **Views**:
  - a 3D dollhouse view, with the walls nearest the camera cut away and the ceilings hidden
  - a 2D plan with dimensions
  - a first-person walkthrough with adjustable eye height and no walking through walls
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
- **Editing**: drag items along the floor with snapping (to a grid, walls and other items), rotate them, raise wall-mounted items, and type exact sizes and positions. Duplicate, delete, hide, lock and group items. Undo and redo are unlimited, and everything works with a mouse, trackpad or keyboard.
- **Surfaces**: wallpaper, tiles, flooring, ceilings, woodwork, and paint per wall or per room. Paint can be set from a colour picker, a hex value, "match this photo", or a named paint colour that you approximate (with a note that screens aren't exact).
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

- I put a photo or screenshot in `private/inbox/` and type `/add-item`. The app's "Add from photo" button also saves a picture there and tells me what to type. I can add a product link, the price, or the dimensions from the listing. If there's a link, read the listed dimensions from the page.
- You identify the object and write an item proposal:
  - category and style
  - width, depth and height in millimetres, with a confidence and how you got them (read from the listing or estimated)
  - materials and colours
  - how it's mounted: floor, wall, ceiling, tabletop or window
  - parameters for the matching parametric model
- In the app, I see a 3D preview next to the photo and correct the size or colours if needed. The item then goes into "My items", ready to place anywhere, as often as I like.
- There are two ways to build the object:
  1. **Parametric** (the default): the matching catalogue model, sized and coloured to match.
  2. **Flat items** (art, photos, posters, mirrors, rugs, wallpaper, tiles, flooring): use the image itself, framed, as a rug, or tiled across a surface. Remove the background on the laptop where it helps.
- I can put several photos in the inbox at once.
- I can add new photos of the house at any time (from our next visit, say) to `private/inputs/` and run `/scan-house` again. You propose corrections to the model.

## Design help

- I ask in the Claude chat. For example: "/design suggest a living room layout with our sofa and a reading corner", "show this bedroom in a Japandi style", or an inspiration photo with "make it feel like this".
- You read the current project, use the catalogue and My items, check clearances with the project's scripts and render views to look at your own result. Then you write a scenario proposal that I preview, accept or reject in the app.

## Milestones

Build in this order. Each milestone ends with the tests passing, the app running locally with one command, `docs/STATUS.md` updated, and a short note telling me what to try.

- **M0, foundations**:
  - Check what's installed on this laptop (Git, Node.js LTS) and help me install anything missing. Ask before installing anything.
  - Set up the scaffold, the `private/` folders and `.gitignore`, the schema, the demo house and the local server.
  - Build a 3D viewer with orbit and plan views, and saving, loading and exporting projects.
  - Add a preview configuration so the app opens in the desktop app's Browser pane.
  - Write a `CLAUDE.md` describing the architecture, commands and the `private/` rules.
- **M1, floorplan to model**: `/scan-house` for the floorplan, the scripts (crop, PDF text, solve), proposals and the review panel, and the measurement review. Walls, openings, floors, ceilings and stairs appear in 3D and 2D. **Pause here** so I can try our real floorplan.
- **M2, photos**: room matching, calibration, the photo-match overlay, heights, finishes and fixtures, plus the "As listed" layer and the next-visit checklist. **Pause here too.**
- **M3, design editor**: the catalogue, editing, surfaces, lighting, Renovate mode, scenarios, quantities and checks.
- **M4, add from photo**: `/add-item`, item proposals, parametric and flat models, My items, the shopping list and the "Add from photo" button.
- **M5, realism**: quality presets, photo mode, daylight through the day and across the seasons, and the view out of the windows.
- **M6, design help**: `/design` and scenario proposals.

Later, if there's time:

- the garden and exterior
- opening the app on a phone or tablet over our home Wi-Fi
- sharing designs with the rest of the household as exported renders and PDFs
- importing a LiDAR room scan (for example from an iPhone Pro) as a high-accuracy measurement source
- a VR walkthrough (WebXR)
- more detailed 3D models of items from an online image-to-3D service, only if I decide it's worth paying for

## Quality bar

- Unit tests for the geometry, unit parsing, the measurement solve and schema migrations.
- End-to-end tests for the core flows (for example with Playwright).
- Test the pipelines against known answers:
  - Generate an estate-agent-style floorplan (as an image and as a PDF) of the demo house.
  - Render synthetic "listing photos" of it from known cameras.
  - Measure how close the scan and calibration get to the true values.
- Look at your own work. Render or screenshot the app and inspect it, especially renders compared against the source photos.
- By the end, this whole story must work on my laptop:
  1. I put our floorplan and photos in `private/inputs/` and type `/scan-house`.
  2. I review a short list of flagged measurements in the app and see our house in 3D.
  3. In photo-match, the living room lines up with the estate agent's photo.
  4. I empty out the seller's furniture, put a photo of our sofa in the inbox and type `/add-item`.
  5. I try two paint colours as two scenarios.
  6. I walk through in first person and render the living room in photo mode.
  7. I check the shopping list total.

## Working agreement

- Work autonomously within these decisions. Ask me only when you're blocked on something that's genuinely my call, such as installing software or anything that costs money or uses an online service.
- Depth over breadth: finish each milestone properly before starting the next.
- Commit locally in small, clear steps, and don't push to GitHub unless I ask. Keep `docs/STATUS.md` current (done, next, known issues) so a fresh session can pick up where you left off.
- Never commit `private/` or real house data.
- Be honest about accuracy. When something is estimated, say so, both in the app and to me.
