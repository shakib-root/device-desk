# Floor plan as isometric pencil-sketch art, one per Area

The Floor Plan is not a plain schematic. Each Area, and the Meeting Rooms, gets its own isometric drawing in a hand-drawn style, made to match the room photos (furniture arrangement, windows, doors, plants). An Office Overview switches between them. An agent draws each picture once as vector art from the photos. Every Desk and Meeting Room is its own element carrying `data-desk-id` / `data-room-id`. The server still renders the page and adds each Space's State as a light wash over that element: green for mine, grey with the Owner's initials for taken, red hatching for blocked, and a dashed outline for partly booked. The drawings are data stored next to `layout.yaml`, so redrawing a room touches no code.

## Considered options

- **A schematic top-down floor plan**, as the plan first described. Rejected: the team wants the map to feel like their own room.
- **Tracing from the photo's own camera angle.** Rejected: desks further back get small and some hide behind others, which makes them hard to tap on a phone.
- **A photo run through an image filter, with hotspots on top.** Rejected: it carries over people, clutter and muddy shading, and colouring a raster image by State is crude.
- **Art made by a person in an outside image tool.** Rejected: it adds a human step every time a room changes.

## Consequences

- Each drawing must keep one element per Space, with the Layout's ID. The Feature Map and tests rely on those IDs, not on positions.
- Area 2 has no photo. Its drawing is based on the human's spoken description, so it matches the arrangement but not the decor; adding a photo later only means redrawing it.
- The drawings always sit on a warm paper background, even when the rest of the app is in dark mode. Recolouring them would double the art work and lose the rooms' real colours.
- An accessible list version of the same Spaces sits next to every picture (round 1, Q12).
