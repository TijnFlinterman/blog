---
layout: default
---

{% raw %}

# Adaptive Squad Formations for Breach-and-Clear AI

I'm a 3rd year student at Breda University of Applied Sciences, my name is Tijn Flinterman and this blog post covers my journey through AI formations, What it is, how I made it and how I used it to make my SWAT-style breach-and-clear system in C++. 

**Table of contents**
- Stack Up: Why Formations?
- Squad leader to Rear Security: Roles and Slots
- Form Up: Formation Types
- On My Mark: The Squad State Machine
- Tight Quarters: Adaptive Corridor Movement
- Drawing the Floor Plan: Authoring Levels with Tiled
- Eyes On: Detection and the Human Element
- Briefing Room: Designer-Facing Tools
- After-Action Report: Common Problems and Solutions
- Debrief

--- 
<video controls width="100%">
  <source src="assets/videos/hero-breach-sequence.mp4" type="video/mp4">
</video>

## Stack Up: Why Formations?

Formations are structured tactical arrangements of groups. Formations offer a solution if you want an organized crowd layout that keeps its shape when navigating obstacles and other terrain instead of a group of independent pathfinders that just happen to walk in the same direction.

To tackle this topic I picked a SWAT breach-and-clear scenario to serve as an overarching contextual narrative to give the viewer/user a more intuitive view of what to expect and its application of it.
This approach also has the added benefit of its real life counterpart also making heavy use of formations when doing its breach-and-clear. From performing a Rolling-T technique narrow and wide to stacking up next to a breach point and then spreading out to the room.

## Squad Leader to Rear Security: Roles and Slots

A formation is a list of "slots", slots are relative to the formation in position and orientation instead of being in world space.

```cpp
struct FormationSlot
{
    glm::vec2 localOffset;
    int priority;
    float facingOffsetDegrees = 0.0f;
};
```
"priority" sets breach order
"facingOffsetDegrees" lets an agent watch a different direction than the group as a whole is heading.

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

That's pretty much all the math for relative position. "facingOffsetDegrees" gets resolved the same way, except you add it to the formation's heading angle instead of rotating an offset. The transform stays this simple no matter how complex the formation gets. The interesting part is which shapes you define and when you switch between them.

## Form Up: Formation Types

At the end of development I had a few distinct shapes, each used in a different context of the breach sequence. Both corridor formations come from the US Army's urban-combat manual, FM 90-10-1 (that's why I spell it as "surpentine").

**Surpentine** and **Rolling-T** are the two corridor-movement formations, and the ones I spent the most time on.

- **Surpentine** is used in narrow corridors: the squad leader anchors the front-center, with two agents close behind him, left and right, looking outward. The rest of the squad follows in a column, ending with a rear guard facing backward.

- **Rolling-T** is used in wide corridors: Two agents move forward and ahead of the squad leader, left and right (the top bar of the "T"), and each covers the *opposite* side of the hallway from the one they are walking on. A squad leader stands behind them (the stem of the "T") and covers the far end of the hallway, looking between the pair. The manual's fourth man provides rear security. My squads are bigger than the manual's four-man team, so the remaining agents trail in a column and the last one acts as the rear guard, facing backward.

--- 
![alt text](assets/images/formation_variants.png)

Instead of hard-coding the shape in C++, formations are authored in a small formation editor, where the position and rotation of every slot is set per squad size, for a narrow and a wide variant. The editor exports the data as JSON, which the game loads at runtime.

```cpp
const auto& editorSlots = m_formationEditor->GetFormation(squadSize, variant);
if (!editorSlots.empty())
{
    std::vector<glm::vec2> localOffsets;
    std::vector<float> localRotations;
    for (const auto& es : editorSlots)
    {
        localOffsets.push_back(glm::vec2(-es.offset.y, -es.offset.x));
        localRotations.push_back(-es.rotationDegrees);
    }
    *m_currentFormation = Formation::CreateFromLocalOffsets(localOffsets, localRotations, FormationType::Corridor);
    AssignFormationSlots();
    return;
}
```

Keeping the exact spacing and angles out of C++ was a something I purposely spent time on to give clearer context of relative positions and rotations (which is important for formations). This made it a lot more intuitive instead of having to make a huge list of magical numbers which are hard to tweak and imagine.

**Wall Stack** lines agents up against the wall besides the breach point. The direction that it stacks on is set in level data. This formation is taken when the squad is close enough to a breach point where they prepare to breach.

**Room Clear Spread** fans agents across an arc once they're through a door, covering the room:

```cpp
Formation Formation::CreateRoomClear(int numAgents, float radius)
{
    Formation formation(FormationType::RoomClearSpread);
    if (numAgents == 1)
    {
        formation.m_slots.push_back({{0.0f, 0.0f}, 0});
        return formation;
    }

    float angleStep = glm::pi<float>() / (numAgents - 1);
    float startAngle = -glm::half_pi<float>();

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
--- 
![Formation types overview](./assets/images/formation-types.gif)

*The four formation shapes, top-down: Surpentine and Rolling-T, Room Clear Spread, Wall Stack.*

## On My Mark: The Squad State Machine

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

The squad travels to a breach point using surpentine or rolling-T dependent on the corridor width, forms up in a Wall Stack beside the door to keep clear from enemies within the room, and waits for the breach command. On that command, the squad leader slices the pie sweeping the room to get as much vision of the room before going in.

The code for "Slicing the pie":

```cpp
glm::vec2 radialDir = geometry2d::RotateCounterClockwise(m_formationFacing, rad);
glm::vec2 rawArcPos = m_slicePivot - radialDir * m_sliceRadius;
glm::vec2 resolvedArcPos = navmesh.FindClosestReachablePoint(squadLeaderBody->GetPosition(), rawArcPos);
```

After slicing the pie, the squad breaches. One at a time the agents flood the room, creating a semicircle arc to clear the room for enemies. If the room is clear or the enemies are arrested the agents move to the next breach point, waiting for the next command.

--- 
![Breach sequence at a single doorway](./assets/images/breach-sequence.gif)

*Forming up, slicing the pie, then entry through the breach point.*

## Tight Quarters: Adaptive Corridor Movement

To make the squad be able to switch between surpentine and rolling-T on the fly I needed a way to detect the width of a corridor to decide which formation it should take. So when the squad is in inside a corridor, it measures the width of said corridor and based on a setting value it will switch formations based on an adjustable width :

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

    if (m_currentFormation->GetType() == desired && variant == m_currentRollingTVariant) return;

    m_currentRollingTVariant = variant;

    // ...rebuild the formation slot data for this variant
}
```

The same idea is used for centering the formation when they navigate through corridors, it finds the centerline by measuring the distance between left and right and setting the overall formation target along that path :

```cpp
glm::vec2 FormationSquad::SnapToCorridorCenterline(const glm::vec2& pos) const
{
    for (const auto& region : m_corridorRegions)
    {
        if (!geometry2d::IsPointInsidePolygon(pos, region)) continue;

        RegionBounds b = ComputeBounds(region);
        glm::vec2 result = pos;
        if (b.Width() < b.Height())
            result.x = (b.minX + b.maxX) * 0.5f;
        else
            result.y = (b.minY + b.maxY) * 0.5f;
        return result;
    }
    return pos;
}
```

--- 
![Corridor width switching](assets/images/snap-to-center.gif)

*The squad reshaping from surpentine to rolling-T as the corridor widens, with corridor center snapping.*


When pathfinding, the "path" it gives back only gives me a few points, and the lines between them can drift toward a wall. So I add more points along the path and snap each one to the middle of the corridor :

```cpp
std::vector<glm::vec2> FormationSquad::ResampleAndSnapPath(const std::vector<glm::vec2>& rawPath) const
{
    if (rawPath.size() < 2) return rawPath;

    std::vector<glm::vec2> result;
    result.push_back(SnapToCorridorCenterline(rawPath.front()));

    for (size_t i = 0; i + 1 < rawPath.size(); ++i)
    {
        glm::vec2 a = rawPath[i];
        glm::vec2 b = rawPath[i + 1];
        float segLen = glm::distance(a, b);
        int steps = std::max(1, static_cast<int>(segLen / m_centerlineResampleSpacing));

        for (int s = 1; s <= steps; ++s)
        {
            float t = static_cast<float>(s) / steps;
            result.push_back(SnapToCorridorCenterline(glm::mix(a, b, t)));
        }
    }
    return result;
}
```

The first version took only one direction across the hallway for centering. This was good enough for the early stages of development when all my hallways were vertical, but it was useless when moving to a horizontal one.
The squad had to find out which direction would be considered "across the hallway" so it now checks the segment "shape" itself.

--- 
![alt text](assets/images/snap_to_center_corner.gif)

*Center snapping working horizontally and vertically.*

## Drawing the Floor Plan: Authoring Levels with Tiled

Levels are built in [Tiled](https://www.mapeditor.org/) instead of a custom in-engine editor I started with and later abandoned. I felt like using this was a lot better for the time investment since I would have been just making a worse version of tiled. The level is spread out in ("Floor", "Corridor", "Walls") to differentiate walkable and non-walkable. 

A point-object layer is used for breach points (entry direction, stack side, visit order) and spawn locations for enemies and agents, breach points read out as plain custom properties:

```cpp
TiledBreachPoint bp;
bp.position = worldPos;
bp.direction = glm::normalize(glm::vec2(
    props.count("dirX") ? props["dirX"] : 1.0f,
    props.count("dirY") ? -props["dirY"] : 0.0f));
bp.order = props.count("order") ? static_cast<int>(props["order"]) : static_cast<int>(result.breachPoints.size());
```

Agents and enemies are simply just stored as 2D vectors :

```cpp
std::vector<glm::vec2> agentSpawns;
std::vector<glm::vec2> enemySpawns;
```

One improvement I made was merging loose tiles into rectangles for better navmesh generation. My first version created one polygon per filled tile and relied on the navmesh generator's polygon-union step to stitch them back into a single walkable region. But when the map expanded to be thousands of tiles big, for a higher resolution map, this became an issue.

To resolve the problem I replaced this with a greedy merge: grow each region right as far as it can, then down as far as the full width stays filled, repeat until every filled cell belongs to some rectangle.

```cpp
for (int row = 0; row < mapHeight; ++row)
{
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
            {
                if (!grid[row + height][c] || visited[row + height][c])
                {
                    canExtend = false;
                    break;
                }
            }
            if (canExtend) ++height;
        }

        for (int r = row; r < row + height; ++r)
        {
            for (int c = col; c < col + width; ++c) 
            {
                visited[r][c] = true;
            }
        }
            
        result.push_back(MakeRectPolygon(col, row, width, height, mapWidth, mapHeight, unitsPerTile));
    }
}
```

Improvement numbers:
- Floor layer from 15,674 tiles to 10 rectangles
- Walls layer from 1,492 tiles to 21 rectangles
- Corridor layer from 9,454 tiles to 4 rectangles

--- 
![Tiled level editor](assets/images/tiled-editor.png)
*The level authored in Tiled: Floor, Corridor, and Walls layers, with a Breach Point object's custom properties visible.*

## Eyes On: Detection and the Human Element

Formation movement alone doesn't make a breach feel tactical — it needs something to react to.

**Vision cones** Each agent carries a `VisionCone` (range, field of view, current target); every frame it checks nearby detectable entities for angle and range, then fires a single raycast to confirm line of sight:

```cpp
            float angle = glm::acos(glm::clamp(glm::dot(cone.facing, dirToTarget), -1.f, 1.f));
            if (angle > halfFovRad) continue;

            const float startOffset = viewerRadius + 0.05f;
            const vec2 rayStart = eyePos + dirToTarget * startOffset;

            const auto& hit = physicsSystem.RaycastGetNearest(rayStart, dirToTarget);

            bool blocked = false;
            if (hit.has_value() && hit->m_rayDistance < (dist - startOffset - 0.1f))
            {
                auto* hitBody = ecs.Registry.try_get<physics::Body>(hit->m_hitEntity);
                blocked = !hitBody || hitBody->GetType() == physics::Body::Static;
            }
            if (blocked) continue;

            cone.hasTarget = true;
            cone.detectedTarget = targetEntity;
            cone.lastKnownPosition = targetPos;

            if (!ecs.Registry.any_of<Spotted>(targetEntity))
            {
                ecs.Registry.emplace<Spotted>(targetEntity);
            }
```

**Fog of war** Rather than tracking visibility per enemy, I track it over a grid of fixed-size cells laid out across the walkable regions, each marked `everSeen` once any agent's vision cone confirms it:

```cpp
for (auto [viewerEntity, cone, viewerBody] : ecs.Registry.view<VisionCone, physics::Body>().each())
{
    int cellRadius = static_cast<int>(std::ceil(cone.range / m_cellSize)) + 1;
    glm::ivec2 centerCoord = WorldToCellCoord(eyePos);

    for (int dy = -cellRadius; dy <= cellRadius; ++dy)
    for (int dx = -cellRadius; dx <= cellRadius; ++dx)
    {
        // ...angle + range check against the cell center, same shape as the vision cone check above

        const vec2 rayStart = eyePos + dir * startOffset;
        auto hit = physicsSystem.RaycastGetNearest(rayStart, dir);
        bool blocked = hit.has_value() && hit->m_rayDistance < (dist - startOffset - 0.1f);
        if (blocked) continue;

        cell.everSeen = true;
        anyCellChangedState = true;
    }
}
```

The unseen-area mesh only gets rebuilt on frames where at least one cell actually changed state `anyCellChangedState`, a small optimization, since most frames nothing new is revealed.


![Vision cones and fog of war](./assets/images/vision-fog.gif)

*Vision cones sweeping as fog of war clears, with a cone turning red the moment it spots an enemy.*

**Arrests and roles** Once spotted, a teammate leaves the squad to arrest an enemy, removing that agent from the squad for the rest of the rooms and formations:

```cpp
entt::entity arrestingAgent = m_swatSquad->GetAgents().back();
if (m_swatSquad->RemoveAgent(arrestingAgent))
{
    ecs.Registry.emplace<ArrestingAgent>(arrestingAgent, nearestEnemy);
    // ...send them toward the enemy's position
}
```

Roles like Squad Leader, Breacher, Support, Rear Guard are stored in "slotIndex" relative to squad size:

```cpp
static const char* SlotToRoleName(int slotIndex, int squadSize)
{
	if (slotIndex == 0) return "Squad Leader";
	if (slotIndex == 1) return "Breacher";
	if (slotIndex == squadSize - 1) return "Rear Guard";
	return "Support";
}
```

This means an arrest, which shrinks the roster and re-indexes everyone, assigns correct role for free. If the agent in slot 1 (Breacher) is pulled off to make an arrest, the remaining agents are re-indexed from 0, and whoever was slot 2 becomes the new slot 1, picking up the Breacher label automatically, with no explicit "who's next" logic anywhere. The rear guard is always the last in the squad, the one with the highest `slotIndex`.

## Briefing Room: Designer-Facing Tools

For usability, I made an ImGui panel that exposes squad speed, breach timing, corridor thresholds, and slice angle/duration as sliders that apply instantly, alongside buttons for every squad command and toggles for each debug layer.

![ImGui tuning panel](./assets/images/imgui-panel.png)

*The live tuning panel: squad state, current formation, every tunable slider, and squad commands.*

A checkpoint system stores the full roster, positions, and formation the moment the squad reaches `ReadyToBreach`:

```cpp
void FormationSquad::SaveReadyToBreachCheckpoint()
{
    auto& ecs = Engine.ECS();
    m_readyCheckpoint.agents = m_agents;
    m_readyCheckpoint.agentPositions.clear();

    for (bee::Entity agentEntity : m_agents)
    {
        auto* body = ecs.Registry.try_get<physics::Body>(agentEntity);
        m_readyCheckpoint.agentPositions.push_back(body ? body->GetPosition() : glm::vec2(0, 0));
    }

    m_readyCheckpoint.formation = std::make_unique<Formation>(*m_currentFormation);
    m_readyCheckpoint.formationCenter = m_formationCenter;
    m_readyCheckpoint.formationFacing = m_formationFacing;
    m_readyCheckpoint.breachPointIndex = m_currentBreachPointIndex;
    m_hasCheckpoint = true;
}
```

This is what lets one button rewind *just the current room's attempt*.

**Designer feedback** I sat down with a system designer and talked through the project with fresh eyes. The biggest takeaway wasn't a specific feature request, it was that how much control a system should expose, and in what shape, depends entirely on who you're building it for. That means actually looking it up, asking, or putting yourself in their shoes, rather than guessing at what feels right to a programmer.

A few concrete things came out of that conversation:

- The formation editor should label slots by name, not number. Reasoning about "the left flanker" shouldn't require remembering that's slot 2.
- A proper pause/scrub control over time would be more valuable than most of the discrete squad commands I built (move to breach, execute breach). If you can pause, resume, and step forward at will, you don't need a separate button for every transition.
- Arrests need more than one fixed outcome. Whether an arresting agent rejoins the squad immediately, escorts the enemy out of the building first and then rejoins, or does something else entirely should be a setting, not a hardcoded behavior.
- An action log with timestamps would make it possible to see exactly how long each phase takes and how timing pressure actually shapes the outcome of a run, rather than eyeballing it.

None of this is implemented yet, but it's a clear list of what I'd tackle next if I kept building on this system.

## After-Action Report: Common Problems and Solutions

**Formation targets that are reachable but tactically wrong.** A slot can land on a navmesh point that is technically valid, but only reachable by walking all the way around a wall instead of taking the short route the formation implies. I tried fixing this with a Dijkstra flood over the navmesh graph, scoring candidate points by real walking distance instead of straight-line distance. It worked in corridors, but it broke the room-clear spread. When the squad breaches a room and spreads out, the walking distance from the door to the final positions inside counted as too long. Those points were rejected, so agents refused to walk to spots they were supposed to take. Walking distance was punishing exactly the positions the room clear needed, so I dropped the idea. For now this stays a known limitation, and I'm not chasing a fully general fix.

**Slicing the pie not revealing much through a thick wall.** The technique sweeps along the door's threshold to progressively reveal more of the room before committing. This was conceptually a strong feature but earlier in development it didn't prove as useful due to my thick walls. I resolved the issue by making the tiled authored map to be much higher in resolution, so the walls could be thinner. One tile used to be around 1 meter, now it’s around 10 cm.

![alt text](assets/images/viewing-angles.png)

**Agents blocking each other on approach to the breach point.** Before I made the "wall stack", agents would bunch up on top of the breach point. This wouldn't make sense tactically, everyone stands in the doorway and in danger of all the dangers inside the room. This also blocked the squad leader from slicing the pie which was another problem caused by this. I resolved the issue by adding the wall stack formation that it when approaching a breach point.

**A squad that could softlock on an unreachable position.** The Room Clear Spread's outer slots sit near the full radius at a steep angle; in a tighter room than tuned for, that position can land outside the walkable navmesh, and that one agent's pathfinding silently fails. Since the squad only advances once every agent reports being in position, this could hang the whole mission on one agent that would never arrive. Now, when a target position can't be reached, the agent moves to the closest point on the navmesh instead and marks its move command as done. That turns a hard failure into a small positional error.

**Fog of war.** This was the most expensive feature I built, and I had to fix it twice. The fog splits the walkable area into a grid of small cells, and a cell stays revealed once an agent's vision cone has seen it.

First, a CPU profile showed `FogOfWar::Update` using **93.6% of all CPU time**, almost all of it in the physics raycast that checks whether each cell is visible. I replaced the raycast with a line-of-sight walk across the fog grid itself, where a missing cell counts as a wall, and I only update the fog ten times a second. That brought it down to **35.4%**.

![CPU profile before: FogOfWar::Update at 93.6% of CPU time, mostly inside the physics raycast](assets/images/fog_profile_1_raycast.png)

*Before: the raycast per cell takes almost all of the CPU time.*

![CPU profile after the grid line of sight: FogOfWar::Update at 35.4%, mostly hash map lookups](assets/images/fog_profile_2_hashmap.png)

*After the line-of-sight walk: the cost moves to hash map lookups.*

Second, a new profile showed the cost had moved: most of what was left was hash map lookups, because I looked up every cell around every agent in an `unordered_map`. Switching to a flat array indexed by cell coordinate brought it down to **about 7%**. What remains is mostly rebuilding the mesh when new cells are revealed.

![CPU profile after the flat array: FogOfWar::Update at about 7%](assets/images/fog_profile_3_flat_array.png)

*After the flat array: what remains is mostly rebuilding the mesh.*

## Debrief

Formations turn out to be mostly discipline in a coordinate transform and local offsets, a shared center and facing, and judgment about when to swap which shape is active. The real work was everywhere around that core: making the group's movement respect the terrain, giving it something to react to, keeping squad-composition changes like an arrest coherent with role assignment and making the whole thing legible enough that someone other than me could sit down and immediately understand what it was doing.

If I keep building on this, the natural next steps are coordinated multi-squad tactics and giving detected threats somewhere to actually retreat to. But as a first dive into formations, this covered the ground I set out to cover.

## Resources

A few resources that helped shape the tactical side of this project:

- [Squad-Based Coordination in the AI of F.E.A.R.](https://www.gdcvault.com/play/1022016/Squad-Based-Coordination-in-The) — GDC talk on coordinated squad AI, directly relevant to how roles and coordination are handled here.
- [Close-quarters battle (Wikipedia)](https://en.wikipedia.org/wiki/Close-quarters_battle) — general background on the real-world tactics (stacking, slicing the pie, staggered entry) this system is modeled after.
- [TC 90-1, Training for Urban Operations](https://armypubs.army.mil/epubs/DR_pubs/DR_a/pdf/web/tc90_1.pdf) — US Army training circular on urban operations training, used as reference for breach and room-clearing procedure.
- [FM 90-10-1, An Infantryman's Guide to Combat in Built-Up Areas](http://pdf.textfiles.com/manuals/MILITARY/united_states_army_fm_90-10x1%20-%2012_may_1993%20-%20part09.pdf) (pages 6, 10–18) — US Army field manual covering room-clearing fundamentals.

![Buas_Logo](./assets/images/Logo_BUas_Black.png)

{% endraw %}