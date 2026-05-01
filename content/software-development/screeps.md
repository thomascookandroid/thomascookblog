+++
date = '2026-04-29T20:10:07+01:00'
title = 'Screeps'
+++

Screeps is this awesome little game which is for programmers to pit their custom developed AI's against one another in a real time strategy game which is ran on a shared shard server.

You can purchase it ![Here](https://store.screeps.com/) if this article interests you, but be warned they apparently had some security issue last year, although I imagine that's been patched now, but fair warning!

Basically, you write Javascript (or Typescript - there are full TS bindings provided) to control Screeps (which are the games units) which spawn in a room, within a larger grid of many rooms known as "the world". Each room has unique randomly generated geometry and resource nodes, and your script needs to have your Screeps sense their environment, then plan and execute actions in order to farm the resources, build a base, including production facilities, storage facilities and defensive structures. Eventually, the aim is to expand to other rooms, which will lead to conflict with other players Screep armies, so you need to code in combat and squad AI too.

The API is fairly well documented, and it's relatively low level; it lets you query rooms for information, and it lets you fetch game objects via ids (such as Screeps, Buildings, walls etc) and it exposes commands for having Screeps move, build, attack etc, but it doesn't provide all the algorithms you need to do pathfinding, geometry searching, base building etc, so you have to write all that from scratch, which is where all the fun is.

It's been a couple years since I last touched my Screep bot, but I went fairly deep into it last time, implementing caching of calculated paths for optimisation, flood fill algorithms to determine the best build locations, manager classes to manage the colony, squad manager to manage squads of attacker Screeps.

![Here](https://github.com/thomascookandroid/Screeps-AI/tree/develop/src) is my Screeps repository containing my Screeps AI in all it's glory.

Here's an implementation of an algorithm that finds the closest "valid" "cell" in a room to the supplied `position` where "valid" is with respect to the supplied heuristic.

```
const findClosestValidRoomPosition = (room, position, heuristic) => {
    const start = room.getPositionAt(position.x, position.y);
    if (start == null)
        return null;
    const frontier = [];
    frontier.push(start);
    const reached = new Set();
    reached.add(start.toString());
    while (frontier !== undefined && frontier.length > 0) {
        const current = frontier.shift();
        const objects = room.lookAt(current.x, current.y);
        const isValid = heuristic(objects);
        if (isValid)
            return current;
        const neighbours = getNeighbours(current);
        for (const neighbour of neighbours) {
            if (!reached.has(neighbour.toString())) {
                frontier.push(neighbour);
                reached.add(neighbour.toString());
            }
        }
    }
    return null;
}
```

This code was responsible for paritioning the supplied cost matrix into regions using a flood fill algorithm. The cost matrix was used to assign a weight to each "cell" in the room representing the cells distance from the nearest wall. Once this matrix was populated, I then need to partition it into "regions" of contigous cells, as this would allow me to select the largest contigous region whose cells were all `x` distance from the nearest wall, in effect a heuristic for "largest buildable area that won't block Screep movements by building against walls.

```
const partitionCostMatrix = (costMatrix, minCost) => {
    const contiguousRegions = [];
    const costMatrixIterator = iterateMatrix(costMatrix);
    const cellsAlreadyAssignedToRegions = new Set();
    for (const cell of costMatrixIterator) {
        if (cellsAlreadyAssignedToRegions.has(cell))
            continue;
        const region = [];
        const visited = new Set();
        const frontier = [];
        frontier.push(cell);
        visited.add(cell);
        while (frontier.length > 0) {
            const current = frontier.shift();
            if (current.v < minCost)
                continue;
            const neighbours = getNeighbours(current);
            for (const neighbour of neighbours) {
                const matrixNeighbour = {
                    x: neighbour.x,
                    y: neighbour.y,
                    v: costMatrix.get(neighbour.x, neighbour.y)
                }
                if (!visited.has(matrixNeighbour)) {
                    visited.add(matrixNeighbour);
                    frontier.push(matrixNeighbour);
                    if (matrixNeighbour.v >= minCost && !cellsAlreadyAssignedToRegions.has(matrixNeighbour)) {
                        region.push(matrixNeighbour);
                        cellsAlreadyAssignedToRegions.add(matrixNeighbour);
                    }
                }
            }
        }
        contiguousRegions.push(region);
    }
    return contiguousRegions;
}
```

I would then cache that returned array of regions in a JSON blob and only re-calculate it when new structures were created to avoid doing costly calculations every game tick as the Screeps server enforces a CPU limit on each tick, effectively forcing you to optimise your code which I thought was super cool.

I also wrote a simple visualiser for my contigous region flood fill, so that I could debug issues with it:

```
visualiseRoomDistanceTransform() {
	const roomDistanceTransform = this.refreshRoomDistanceTransform(this.room);
	const iterator = iterateMatrix(roomDistanceTransform);
	for (const cell of iterator) {
		this.room.visual.text(cell.v, cell.x, cell.y);
	}
}
```
