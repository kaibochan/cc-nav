# cc-nav

**Overview**
- **Purpose:** Autonomous 3D mapping and pathfinding for a ComputerCraft turtle using GPS and the turtle API.
- **File:** [nav.lua](nav.lua)

**What problem this solves**
- Allows a turtle to discover and persist a 3D local map, plan an efficient route to a target position, and execute movement (including turning and vertical moves) with optional mining and remote control.

**Main features**
- **Mapping & persistence:** `setBlock`, `getBlock`, `saveMap`, `loadMap`, `mergeMap` — stores map as a nested table `map[x][y][z] = blockname`.
- **Sensing:** `inspect`, `inspectUp`, `inspectDown`, `inspectAll`, `inspectAllQuick` to populate the map.
- **Calibration:** `calibrate()` uses `gps.locate()` and a short move to determine `pos` and `dir`.
- **Movement primitives:** `move`, `moveBack`, `moveUp`, `moveDown`, `turnLeft`, `turnRight`.
- **Pathfinding:** `findPath`, `compilePathInstructions`, `genAdjMoveInstr` — A*-style search with Manhattan heuristic.
- **Mining control:** `setMineMode`, `useWhitelist`, `useBlacklist`, `mineWhitelist`, `mineBlacklist`.
- **Region restriction:** `setRestriction` / `clearRestriction` and `isWithinRegion`.
- **Remote commands:** `listen` receives `stop`, `go`, `where`, and debug queries via a modem peripheral.

**Key data structures**
- **Map:** nested table indexed by normalized coordinates (0 remapped to `2147483647`) — `map[x][y][z] = blockname`.
- **`pathNode` objects:** hold `position`, `parent`, `direction`, `distFromTarget`, `runningCost`, and per-node `instructions` (metatable `instructions_meta`).
- **Open set (`toBeProcessed`):** a sorted array of `pathNode` items acting as the frontier/priority queue.
- **`instructions`:** list of action functions (e.g., `move`, `turnRight`) with a callable metatable that executes them and returns success & last successful index.

**Algorithms & heuristics**
- **A*-style search:**
  - Heuristic: taxicab (Manhattan) distance via `getDist(p1, p2)`.
  - Running cost: cumulative action count (`runningCost`) where marginal cost = number of instructions needed to move from parent to child.
  - Node ordering: comparator in `pathNode_meta.__lt` compares `score()` (runningCost + heuristic) and biases toward nodes with known map data.
  - Expansion: `findAdjNodes` checks six cardinal neighbors (including vertical moves) and filters by `isNavigable` or mineable status.
- **Instruction compilation:** `compilePathInstructions` concatenates per-node action lists into a single executable instruction sequence.

**Design patterns & approaches**
- **Metatables as lightweight classes:** `instructions_meta` and `pathNode_meta` provide methods and operator overloads (`__call`, `__lt`, `__eq`, `__tostring`).
- **Actions as first-class functions:** Movement and turn operations are stored as function references, separating planning from execution.
- **Event-driven concurrency:** `parallel.waitForAny(gotoGlobalTarget, listen)` runs navigation and modem listener concurrently.
- **Coroutines & cooperative yielding:** `yeild()` queues an empty event and yields to allow other coroutines to run during search/execution loops.
- **Configurable behavior:** global flags (`mineMode`, `restrictMode`, `useMineWhitelist`) adjust runtime behavior without changing core algorithms.

**Notable implementation details & examples**
- Instruction generation for adjacent moves: `genAdjMoveInstr(p1, d, p2)` returns a list of actions (turns + `move` or `moveUp`/`moveDown`).
- Node running-cost update: `node.calculateRunningCost()` uses `genAdjMoveInstr` and updates `node.runningCost = node.parent.runningCost + node.marginalCost()`.
- Priority insertion: `addNode(adj, nodes, toBeProcessed, target)` inserts `adj` into `toBeProcessed` in sorted order using `adj < toBeProcessed[i]`.
- Execution of compiled instructions: `instructions_meta.__call` runs every action and returns `(success, lastSuccess)`.

**Usage (high-level)**
- Calibrate position & orientation: call `calibrate()` (requires GPS).
- Start pathfinding to a target: `pathFind(vector.new(x, y, z), dist)` — runs until arrival within `dist` or a `stop` command.
- Toggle mining: `setMineMode(true)` and control whitelist/blacklist via `useWhitelist()` / `useBlacklist()`.

**Assumptions & operational notes**
- Depends on ComputerCraft APIs: `turtle`, `gps`, `peripheral`, `modem`, `textutils`, `fs`, `parallel`.
- Expects a modem on the left (`peripheral.wrap("left")`).
- Uses sentinel `2147483647` to normalize coordinate `0` for table indexing.
