# Disaster Rescue Coordination System

Data Structures PBL, written in C. Standard library only, no graphics library.
Runs in a normal terminal with a menu and a live ASCII city map.

## Build

```bash
gcc main.c dispatch.c heap.c queue.c hash.c graph.c render.c -o rescue
```

or just `make`. Then run `./rescue` (`rescue.exe` on Windows).

Test drivers:

```bash
make tests
```

## Files

| File | Owner | What is inside |
|---|---|---|
| common.h | All (D drafted) | MAX_NODES, MAX_AMB, TABLE_SIZE, ROWS, COLS, Request, Victim, status enum |
| heap.h / heap.c | B | max-heap of requests + the min-heap Dijkstra borrows |
| queue.h / queue.c | A | circular queue of free ambulances |
| hash.h / hash.c | A | victim table, chaining |
| render.h / render.c | A | canvas, node_pos[], draw_edge, map_draw |
| graph.h / graph.c | C | adjacency list, dijkstra, get_path, print_path, nearest_center |
| dispatch.c | D | menu actions, the dispatch cycle, input validation, demo data |
| main.c | D | menu loop |

Test drivers: test_queue.c, test_heap.c, test_hash.c, test_render.c, test_graph.c

## The demo city

15 junctions (0 to 14) in 3 rows of 5, joined by 22 two-way roads.
Rescue centres are at junctions 0, 9 and 11.
Road weights are travel minutes.

```
   [R]------4------(1)------6------(2)------5------(3)-----7-----(4)
    |               |               |               |             |
    9               2               5               6             8
    |               |               |               |             |
   (5)------3------(6)------8------(7)------2------(8)-----6-----[R]
    |               |               |               |             |
    4               7               3               4             5
    |               |               |               |             |
  (10)------5------[R]------4-----(12)------9-----(13)-----3----(14)
```

## Design choices we had to pick (for the report)

**Circular queue, full vs empty.** We use a `count` variable, not the
sacrifice-one-slot trick. So `q_is_empty()` is `count == 0` and `q_is_full()` is
`count == MAX_AMB`, and all MAX_AMB slots are usable. Option 6 prints the front
and rear index so the wrap around is visible on screen.

**Heap priority key.** `better()` in heap.c is the only place where order is
decided. Higher severity first, and if two requests have the same severity the
smaller (earlier) timestamp wins. That is why a minor request from long ago is
still served before a minor request that just came in, and why nothing starves.
`hp_pop` moves the LAST element to the root first, so the tree stays complete,
and then sifts down.

**TABLE_SIZE is 11, a prime.** With a prime size, ids that share a common factor
still spread over the buckets instead of piling into a few. Collisions are
handled by chaining, so a bucket is a small singly linked list. 4001 and 4012
both land in bucket 8 on purpose in the demo data, to show the chain.

**Adjacency list, not a matrix.** Each junction touches only 3 or 4 roads, so a
15x15 matrix would be mostly zeros. The list is O(V+E).

**Lazy deletion in Dijkstra.** We never implement decrease-key. We just push a
better distance again, and when a node pops a second time `done[]` is already 1
so we skip it. That popped entry is the stale one.

**Screen coordinates are not weights.** `node_pos[]` lives in render.c and
nothing else reads it. Dijkstra only sees the weights in graph.c. That is why a
road that looks short on screen can still cost 9 minutes (damaged road, slow
traffic). The two are kept in separate files on purpose.

**The ambulance stays out until it reports back.** Dispatch marks the victim
DISPATCHED and the ambulance is gone from the free ring. Option 3 brings it
back, flips the victim to RESCUED and re-enqueues the ambulance at the rear. If
we re-enqueued straight away the "no free ambulance" case could never happen and
that whole branch would be dead code.

**No free ambulance means peek, not pop.** If the ring is empty we call
`hp_top()` and leave the request on the heap. Popping it would throw the most
critical request away.

## How to show it in the viva

1. Option 9, load the 6 demo victims.
2. Option 5, show the pending heap. Index 0 is the critical one even though it
   was not the first one typed.
3. Option 2 five times. Watch the critical ones leave first and the route light
   up with `#` on the map each time.
4. Option 6. Zero free, all 5 out, and the front/rear indexes have moved.
5. Option 2 again. It refuses and says the request was NOT taken off the heap.
6. Option 5 again to prove the request is still sitting there.
7. Option 3, bring AMB-2 back. Rear index wraps around to 0.
8. Option 2. Now the last victim gets AMB-2.
9. Option 4 on victim 4002, and option 8 for the whole table with buckets.

## Complexity

| Operation | Structure | Time |
|---|---|---|
| register request | max-heap push | O(log n) |
| pick most critical | max-heap pop | O(log n) |
| get / free ambulance | circular queue | O(1) |
| victim lookup / update | hash table, chaining | O(1) average, O(n) worst |
| fastest route | Dijkstra + min-heap | O((V+E) log V) |
| redraw the map | 2D canvas | O(ROWS*COLS + E) |

One full dispatch cycle is O((V+E) log V) because the routing dominates.

## Not done

The optional extensions from the spec are not implemented: blocked roads with
re-routing, centre capacity limits, saving to a file, and the ambulance
animation along the route.
