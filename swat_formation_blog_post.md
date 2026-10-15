---
layout: default
---

# Adaptive Squad Formations for Breach-and-Clear AI

This blog post is an introduction to formations in game AI — what they are, the building blocks they're made of, and how I used them to build a SWAT-style breach-and-clear system in C++ for a project as a third-year student at Breda University of Applied Sciences.

**Table of contents**
- Why Formations?
- The Building Blocks
- Formation Types
- The Squad State Machine
- Adaptive Corridor Movement
- Authoring Levels with Tiled
- Detection and the Human Element
- Designer-Facing Tools
- Common Problems and Solutions
- Conclusion

## Why Formations?

Squad-based games — RTS titles, tactical shooters, pack AI in racing games — need a group of agents to move and act as a cohesive unit rather than a pile of independent pathfinders that happen to be heading the same direction. A formation gives you that: a shared reference frame (a center point, a facing direction) that individual agents position themselves relative to.

I picked a SWAT breach-and-clear scenario to learn this with, because it forces most of the interesting formation problems to show up on their own: tight single-file movement through corridors, a wider spread when clearing a room, a specific stacked shape at a doorway before entry, a staggered sequence for going through that door. If a formation system handles all of that cleanly, it handles most things a squad-based game would ask of it.

## The Building Blocks

A formation is a list of **slots** — positions expressed *relative* to the formation's own center and facing, rather than in absolute world space:

```cpp
struct FormationSlot
{
    glm::vec2 localOffset;
    int priority;
    float facingOffsetDegrees = 0.0f; // relative to the formation's own facing
};
```

`priority` sets breach order (slot 0 enters first); `facingOffsetDegrees` lets a slot watch a different direction than the group as a whole is heading, which matters for movement formations where some agents should be watching the flanks rather than all facing forward.

Converting a slot into a world position is a rotation and a translation — rotate the local offset by the formation's current facing angle, add it to the center:

```cpp
glm::vec2 Formation::GetSlotWorldPosition(int slotIndex, const glm::vec2& center, const glm::vec2& facing) const
{
    const FormationSlot& slot = m_slots[slotIndex];
    float angle = std::atan2(facing.y, facing.x);
    glm::vec2 rotated = geometry2d::RotateCounterClockwise(slot.localOffset, angle);
    return center + rotated;
}
```

That's genuinely most of the core math — the `facingOffsetDegrees` field resolves to world space the same way, just added to the formation's heading angle rather than rotating an offset. Everything interesting about a formation system comes from *what shapes you define* and *when you switch between them*, not from the transform itself, which stays this simple no matter how elaborate the shape gets.

## Formation Types

I ended up with a handful of distinct shapes, each suited to a specific moment in a breach.

**Column** is the simplest — a single-file line trailing directly behind the direction of travel, used as a fallback when nothing more specific applies.

**Wall Stack** lines agents up flush against a wall beside a doorway, offset to whichever side the level data specifies, oriented to the door's entry direction — this is the shape the squad holds in while waiting for a breach command.

**Room Clear Spread** fans agents across an arc once they're through a door, covering the room rather than bunching in the entryway:

```cpp
Formation Formation::CreateRoomClear(int numAgents, float radius, float angleRangeDegrees)
{
    Formation formation(FormationType::RoomClearSpread);
    if (numAgents == 1)
    {
        formation.m_slots.push_back({ {0.0f, 0.0f}, 0 });
        return formation;
    }

    float angleRangeRad = glm::radians(angleRangeDegrees);
    float angleStep = angleRangeRad / (numAgents - 1);
    float startAngle = -angleRangeRad * 0.5f;

    for (int i = 0; i < numAgents; ++i)
    {
        float angle = startAngle + i * angleStep;
        FormationSlot slot;
        slot.localOffset = glm::vec2(std::cos(angle) * radius, std::sin(angle) * radius);
        slot.priority = i;
        formation.m_slots.push_back(slot);
    }
    return formation;
}
```

The angle range is tunable rather than fixed at 180° for a concrete reason: a full semicircle pushes the outermost agents directly sideways at the full radius, which clips a wall in a tighter room than the formation was tuned for. Narrowing the range (and the radius) made the spread reliably fit inside smaller rooms, at the cost of slightly less coverage at the edges.

**Rolling T (Narrow / Wide)** is the corridor-movement formation, and the one I spent the most time on. In a narrow corridor, agents move single-file but alternate their facing left and right down the line — lead watches forward, last watches rear, everyone between splits attention across the two flanks. In a wide corridor, the squad splits into two lanes side by side instead. Rather than hardcoding either shape in C++, I authored both variants' exact offsets and rotations externally through a small formation editor, and convert that data into slots at runtime:

```cpp
const auto& editorSlots = m_formationEditor->GetFormation(squadSize, variant);
std::vector<glm::vec2> localOffsets;
std::vector<float> localRotations;
for (const auto& es : editorSlots)
{
    localOffsets.push_back(glm::vec2(-es.offset.y, -es.offset.x));
    localRotations.push_back(-es.rotationDegrees);
}
*m_currentFormation = Formation::CreateFromLocalOffsets(localOffsets, localRotations, FormationType::RollingT);
```

Keeping the exact spacing and angles out of hardcoded C++ mattered for iteration speed specifically — nudging a number in the editor and watching the squad react immediately is a tighter loop than changing a constant and recompiling.

## The Squad State Machine

The whole breach sequence is one state machine per squad:

```cpp
void FormationSquad::Update(const Navmesh& navmesh, float dt)
{
    switch (m_state)
    {
        case SquadState::MovingToBreachPoint: UpdateMovingToBreach(navmesh, dt); break;
        case SquadState::FormingUp:           UpdateFormingUp(navmesh, dt);      break;
        case SquadState::SlicingThePie:       UpdateSlicingThePie(navmesh, dt);  break;
        case SquadState::Breaching:           UpdateBreaching(navmesh, dt);      break;
        case SquadState::RoomClearing:        UpdateRoomClearing(navmesh, dt);   break;
    }
    // ...each agent recomputes its path and velocity toward its current slot goal
}
```

The squad travels to a breach point (reshaping its formation to fit the corridor along the way — next section), forms up in a Wall Stack beside the door, and waits in `ReadyToBreach` for an explicit command. On that command, the point man slices the pie — arcing around the door's pivot to peek the room before committing, resolved against the navmesh rather than trusting the raw geometric arc:

```cpp
glm::vec2 radialDir = geometry2d::RotateCounterClockwise(m_formationFacing, rad);
glm::vec2 rawArcPos = m_slicePivot - radialDir * m_sliceRadius;
glm::vec2 resolvedArcPos = navmesh.FindClosestReachablePoint(pointManBody->GetPosition(), rawArcPos);
```

Then the squad breaches: agents enter one at a time on a short delay, each heading straight for its *own* final Room Clear Spread position rather than funneling through one shared point first — a detail that mattered more than I expected, covered below. Once a room is clear, the squad advances to the next breach point in an ordered sequence, repeating the cycle until the last one is done.

## Adaptive Corridor Movement

The part I'm most satisfied with: the formation shape adapts to the level geometry it's actually moving through. Each frame while traveling, the squad checks whether its center is inside a corridor region and, if so, measures that corridor's width, switching between Rolling T Narrow and Wide based on a tunable threshold:

```cpp
void FormationSquad::UpdateFormationShapeForTerrain()
{
    bool inCorridor = IsInCorridor(m_formationCenter);
    FormationEditor::Variant variant = FormationEditor::Wide;

    if (inCorridor)
    {
        float width = MeasureCorridorWidth(m_formationCenter);
        variant = (width < m_narrowWidthThreshold) ? FormationEditor::Narrow : FormationEditor::Wide;
    }

    if (m_currentFormation->GetType() == FormationType::RollingT && variant == m_currentRollingTVariant)
        return; // already the right shape, nothing to do

    // ...rebuild the formation from the externally-authored Rolling T slot data for this variant
}
```

The path itself gets the same treatment, not just the formation shape — raw navmesh paths are resampled into smaller steps, and each point snapped to the centerline of whichever corridor region it falls inside:

```cpp
glm::vec2 FormationSquad::SnapToCorridorCenterline(const glm::vec2& pos, const glm::vec2& travelDirection) const
{
    for (const auto& region : m_corridorRegions)
    {
        if (!geometry2d::IsPointInsidePolygon(pos, region)) continue;

        RegionBounds b = ComputeBounds(region);
        glm::vec2 result = pos;

        if (std::abs(travelDirection.x) >= std::abs(travelDirection.y))
            result.y = (b.minY + b.maxY) * 0.5f; // moving along X -> center across Y
        else
            result.x = (b.minX + b.maxX) * 0.5f; // moving along Y -> center across X

        return result;
    }
    return pos;
}
```

The important detail is `travelDirection` is computed **per path segment**, not once for the whole path:

```cpp
glm::vec2 segDir = (segLen > 0.0001f) ? (segDelta / segLen) : lastDir;
// ...
result.push_back(SnapToCorridorCenterline(p, segDir));
```

My first version used one fixed direction for the entire centering pass, which worked fine down a straight hallway and broke at every turn — a path bending from a vertical corridor into a horizontal one needs to re-decide which axis counts as "across the hallway" exactly at that turn, not carry the old corridor's orientation past the bend. Deriving direction from the segment itself, rather than from any single global value, fixed it everywhere at once instead of case-by-case.

## Authoring Levels with Tiled

Levels are built in [Tiled](https://www.mapeditor.org/) rather than a custom in-engine editor I started with and later abandoned — a grid tool built for exactly this is faster to iterate in than anything I'd build myself in the time available. Three tile layers (`Floor`, `Corridor`, `Walls`) define walkable space and obstacles; a point-object layer carries breach points (entry direction, stack side, visit order) and spawn locations, each read out as plain custom properties:

```cpp
TiledBreachPoint bp;
bp.position = worldPos;
bp.direction = glm::normalize(glm::vec2(
    props.count("dirX") ? props["dirX"] : 1.0f,
    props.count("dirY") ? -props["dirY"] : 0.0f));
bp.order = props.count("order") ? static_cast<int>(props["order"]) : static_cast<int>(result.breachPoints.size());
```

The one piece worth real attention is how tile data becomes navmesh geometry. My first version created one polygon per filled tile and relied on the navmesh generator's polygon-union step to stitch them back into a single walkable region — and on my current level that means tens of thousands of individual tiles. It mostly worked, but meant a large amount of unnecessary unioning, and occasionally left a region looking visibly split even though every tile was painted correctly in Tiled. I replaced this with a greedy merge: grow each region right as far as it can, then down as far as the full width stays filled, repeat until every filled cell belongs to some rectangle:

```cpp
for (int row = 0; row < mapHeight; ++row)
for (int col = 0; col < mapWidth; ++col)
{
    if (!grid[row][col] || visited[row][col]) continue;

    int width = 1;
    while (col + width < mapWidth && grid[row][col + width] && !visited[row][col + width]) ++width;

    int height = 1;
    bool canExtend = true;
    while (canExtend && row + height < mapHeight)
    {
        for (int c = col; c < col + width; ++c)
            if (!grid[row + height][c] || visited[row + height][c]) { canExtend = false; break; }
        if (canExtend) ++height;
    }

    for (int r = row; r < row + height; ++r)
        for (int c = col; c < col + width; ++c) visited[r][c] = true;

    result.push_back(MakeRectPolygon(col, row, width, height, mapWidth, mapHeight, unitsPerTile));
}
```

It's not an optimal rectangle decomposition — that's a more involved problem on its own — but on my current level it took the floor layer from 15,674 raw tiles down to 10 rectangles, walls from 1,492 tiles to 21 rectangles, and the corridor layer from 9,454 tiles to just 4. Beyond the navmesh-generation speedup, that's the fix for the splitting: a region that's one rectangle, or a small handful of them, has no internal seam for a union step to get wrong in the first place.

## Detection and the Human Element

Formation movement alone doesn't make a breach feel tactical — it needs something to react to.

**Vision cones.** Each agent carries a `VisionCone` (range, field of view, current target); every frame it checks nearby detectable entities for angle and range, then fires a *single* raycast to confirm line of sight, rather than sweeping many rays to approximate the same check:

```cpp
float angle = glm::acos(glm::clamp(glm::dot(cone.facing, dirToTarget), -1.f, 1.f));
if (angle > halfFovRad) continue; // outside the FOV wedge

const auto& hit = physicsSystem.RaycastGetNearest(rayStart, dirToTarget);
bool blocked = hit.has_value() && hit->m_hitEntity != targetEntity
               && hit->m_rayDistance < (dist - startOffset - 0.1f);
if (blocked) continue;

cone.hasTarget = true;
cone.facing = dirToTarget; // turn to actually look at what was spotted
```

`rayStart` is offset outward from the agent's own position by its collider radius before the raycast is even fired — starting the ray exactly at an agent's own center meant it would immediately hit its own collider at `t = radius`, before ever reaching a real target, silently failing every detection check. More on that below.

**Fog of war.** Rather than tracking visibility per enemy, I track it over a grid of fixed-size cells laid out across the walkable regions, each marked `everSeen` once any agent's vision cone confirms it:

```cpp
for (auto [viewerEntity, cone, viewerBody] : ecs.Registry.view<VisionCone, physics::Body>().each())
{
    glm::ivec2 centerCoord = WorldToCellCoord(viewerBody.GetPosition());
    int cellRadius = static_cast<int>(std::ceil(cone.range / m_cellSize)) + 1;

    for (int dy = -cellRadius; dy <= cellRadius; ++dy)
    for (int dx = -cellRadius; dx <= cellRadius; ++dx)
    {
        // ...angle + range check against the cell center, same shape as the vision cone check above

        const vec2 rayStart = eyePos + dir * startOffset; // same self-intersection fix as vision cones
        auto hit = physicsSystem.RaycastGetNearest(rayStart, dir);
        if (hit.has_value() && hit->m_rayDistance < (dist - startOffset - 0.1f)) continue;

        cell.everSeen = true;
    }
}
```

Having already hit the self-intersecting raycast bug once with vision cones, fog of war reuses the exact same radius-offset fix from the start rather than rediscovering it. The unseen-area mesh only gets rebuilt on frames where at least one cell actually changed state (`anyCellChangedState`), rather than every frame regardless — a small but free optimization, since most frames nothing new is revealed.

**Arrests and roles.** Once spotted, the squad can peel a teammate off to arrest an enemy, removing that agent from the active roster for the rest of the room:

```cpp
bee::Entity arrestingAgent = m_swatSquad->GetAgents().back();
if (m_swatSquad->RemoveAgent(arrestingAgent))
{
    ecs.Registry.emplace<ArrestingAgent>(arrestingAgent, nearestEnemy);
    // ...send them toward the enemy's position
}
```

Roles (Point Man, Breacher, Support, Rear Guard) aren't stored per agent — they're read off current slot index relative to squad size:

```cpp
static const char* SlotToRoleName(int slotIndex, int squadSize)
{
    if (slotIndex == 0) return "Point Man";
    if (slotIndex == 1) return "Breacher";
    if (slotIndex == squadSize - 1) return "Rear Guard";
    return "Support";
}
```

Which means an arrest, which shrinks the roster and re-indexes everyone, gets correct role promotion for free. If the agent in slot 1 (Breacher) is pulled off to make an arrest, the remaining agents are reindexed from 0, and whoever was slot 2 becomes the new slot 1 — picking up the Breacher label automatically, with no explicit "who's next" logic anywhere.

## Designer-Facing Tools

A live ImGui panel exposes squad speed, breach timing, corridor thresholds, and slice angle/duration as sliders that apply instantly, alongside buttons for every squad command and toggles for each debug layer.

A checkpoint system snapshots the full roster, positions, and formation the moment the squad reaches `ReadyToBreach`:

```cpp
void FormationSquad::SaveReadyToBreachCheckpoint()
{
    m_readyCheckpoint.agents = m_agents; // the full roster, not just positions
    for (bee::Entity agentEntity : m_agents)
    {
        auto* body = Engine.ECS().Registry.try_get<physics::Body>(agentEntity);
        m_readyCheckpoint.agentPositions.push_back(body ? body->GetPosition() : glm::vec2(0, 0));
    }
    m_readyCheckpoint.formation = std::make_unique<Formation>(*m_currentFormation);
    m_readyCheckpoint.breachPointIndex = m_currentBreachPointIndex;
    m_hasCheckpoint = true;
}
```

This is what lets one button rewind *just the current room's attempt*. Deciding what "rewind" should actually mean once an arrest has happened took a moment of real thought: it could restore only positions and leave the roster permanently smaller (treating an arrest as irreversible attrition even on retry), it could refuse to restore at all once an arrest has happened (flagging the checkpoint stale), or it could fully undo the arrest — bringing the agent back and reverting the enemy's cuffed state — so a retry is a genuine clean slate. I went with the third option, since "retry this room" reads to a player as "as if that attempt never happened," which an arrest is very much part of. The detail that makes it possible is storing the full agent list in the checkpoint rather than only positions — a checkpoint that only remembered positions would restore the wrong roster size the instant an arrest had happened since it was saved.

## Common Problems and Solutions

**Agents piling up at the doorway.** My first breach sequence sent every releasing agent toward the same single point a short distance through the door, meaning to let them spread out once inside. In practice this meant agents converging on one coordinate and jamming against each other in the doorway — whoever was released last had to shove through a growing crowd, and in a narrow door simply couldn't. Fixed by computing the full Room Clear Spread *before* releasing anyone, so each agent could be sent directly to its own final slot as it's released.

**A vision cone that could never see anything.** The very first version of the detection raycast started exactly at the viewing agent's own position. A ray starting at the center of a circle immediately re-intersects that same circle's boundary at `t = radius` — so the raycast would report a "blocking" hit (its own collider) before ever reaching a real target, for virtually every detection check regardless of distance. Offsetting the ray's starting point outward by the agent's own collider radius first fixed it — and the same fix got reused directly when fog of war needed the same kind of raycast later.

**A navmesh that looked fine but wasn't.** One polygon per tile, relying on a union step to stitch them together, occasionally left visible splits even in correctly-painted rooms. Merging same-type tiles into rectangles up front fixed it, and incidentally made navmesh generation faster besides.

**A squad that could softlock on an unreachable position.** The Room Clear Spread's outer slots sit near the full radius at a steep angle; in a tighter room than tuned for, that position can land outside the walkable navmesh, and that one agent's pathfinding silently fails. Since the squad only advances once every agent reports being in position, this could hang the whole mission on one agent that would never arrive. A timeout — proceed anyway after a few seconds, with a logged warning — turned a hard failure into a graceful one.

**Two divergent copies of "spawn a squad."** Once restart existed, I had one spawn function from startup and a second, separately-written one for restarts, and they drifted apart in agent speed and vision parameters. A restart would silently create a second full set of agents on top of the still-alive originals; the overlapping colliders got violently flung apart by collision resolution the instant physics ran, which looked exactly like agents vanishing a frame after spawning. Fixed by making exactly one function define what spawning a squad means, called identically from startup and every restart path.

## Conclusion

Formations turn out to be mostly discipline in a coordinate transform — local offsets, a shared center and facing, and judgment about when to swap which shape is active — rather than anything mathematically deep. The real work was everywhere around that core: making the group's movement respect the terrain, giving it something to react to, keeping squad-composition changes like an arrest coherent with role assignment and checkpointing instead of fighting them, and making the whole thing legible enough that someone other than me could sit down and immediately understand what it was doing.

If I keep building on this, the natural next steps are coordinated multi-squad tactics and giving detected threats somewhere to actually retreat to. But as a first dive into formations, this covered the ground I set out to cover.

## Resources

A few resources that helped shape the tactical side of this project:

- [Squad-Based Coordination in the AI of F.E.A.R.](https://www.gdcvault.com/play/1022016/Squad-Based-Coordination-in-The) — GDC talk on coordinated squad AI, directly relevant to how roles and coordination are handled here.
- [Close-quarters battle (Wikipedia)](https://en.wikipedia.org/wiki/Close-quarters_battle) — general background on the real-world tactics (stacking, slicing the pie, staggered entry) this system is modeled after.
- [TC 90-1, Training for Urban Operations](https://armypubs.army.mil/epubs/DR_pubs/DR_a/pdf/web/tc90_1.pdf) — US Army training circular on urban operations training, used as reference for breach and room-clearing procedure.
- [FM 90-10-1, An Infantryman's Guide to Combat in Built-Up Areas](http://pdf.textfiles.com/manuals/MILITARY/united_states_army_fm_90-10x1%20-%2012_may_1993%20-%20part09.pdf) (pages 6, 10–18) — US Army field manual covering room-clearing fundamentals.

![Buas_Logo](./assets/images/Logo_BUas_Black.png)