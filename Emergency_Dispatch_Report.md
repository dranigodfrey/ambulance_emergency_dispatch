# Improving Emergency Response in Arua City Through Efficient Algorithms

**Group Project Report** | Muni University | Advanced Algorithms and Complexity

**Group members:**
| Name	| Registration Number | 
| --- | --- |
| Drani Godfrey | 2026/U/MCS/02228 | 
| Jakuma Kelvin	| 2026/U/MCS/00060 |
| Yokoko Christine| 2026/U/MCS/00065 |
| Apangubo Patrick | 2026/U/MCS/00055 |


---

## 1. Introduction and problem formulation

When someone in Arua calls for an ambulance, a dispatcher has to make three quick decisions: which waiting call to deal with first, which ambulance to send, and which road it should take. Getting any of these wrong costs minutes, and for a cardiac arrest, a difficult delivery or a serious injury, minutes can decide whether the patient recovers. This report describes a Python prototype that helps with these decisions and shows how well it works in experiments.

We chose to treat the problem as one pipeline, not five separate exercises. A call comes in, it is ranked against the other waiting calls, an ambulance is picked, a route is worked out, and then we measure how it all went. In algorithmic terms, the road network is a weighted undirected graph, the waiting calls form a priority queue, choosing an ambulance is a "find the nearest target" shortest-path problem, and routing is an ordinary single-pair shortest-path problem. Framing it this way meant we could rely on well-known algorithms with proven guarantees rather than inventing rules of thumb.

We set the prototype three goals:

1. Serve the most urgent patients first, without letting anyone wait forever.
2. Always send the ambulance that can reach the patient fastest.
3. Find provably optimal routes at a cost that stays reasonable as the city grows.

Everything in this report (the code, the data and the experiments) can be reproduced from the accompanying notebook, because all random seeds are fixed.

## 2. Data and data representation

### 2.1 Dataset

We could not find any public data on ambulance operations in Arua, so we built a simulated dataset that behaves like a real one. The road network has 120 junctions and 209 road segments on a slightly jittered grid. Every fourth row and column is an arterial road (40 km/h) and the rest are local streets (25 km/h). Each road gets:

- a random congestion factor between 1.0 and 1.5;
- a winding factor, so a road is never shorter than the straight line between its two ends;
- for local roads only, a 12% chance of being removed altogether, which imitates dead ends and blocked streets.

We then keep only the largest connected piece of the network, so every junction can be reached from every other.

On top of the network we placed five hospitals, four ambulance stations and eight ambulances (two per station). Sixty emergency calls arrive over a four-hour window following a Poisson process, the standard model for random, independent arrivals. Each call has a severity from 1 to 5, drawn with probabilities of 15%, 25%, 30%, 20% and 10%, and a category such as road traffic accident, maternal emergency or cardiac event.

Two caveats. The hospital names and coordinates are placeholders, and should be replaced with surveyed data if we can get it. The code does not need to change when that happens. The data is stored in six CSV files: nodes, roads, hospitals, stations, ambulances and emergencies.

### 2.2 Choice of data structures

We store the road network as an adjacency list, written in Python as a dictionary of dictionaries. Each junction maps to its neighbours, and each neighbour maps to the travel time along that road. This matches how the algorithms use the graph: they keep asking "who are this node's neighbours?", which the structure answers in time proportional to the node's degree, and looking up a single road takes constant time.

The alternative, an adjacency matrix, needs V² cells. That is 14,400 cells for our small city but 25 million for a city of 5,000 junctions, and nearly all of them would be empty because a typical junction has only about 3.5 roads. The adjacency list needs just V + E entries, so its memory grows in step with the city.

Coordinates sit in a separate dictionary, since only A\* and the map drawings need them. Edge weights are travel times in minutes, not kilometres. Dispatchers care about time, and a short unpaved road can easily be slower than a longer tarmac one.

## 3. Emergency prioritisation

Waiting calls are kept in a binary heap. Inserting a call and removing the most urgent one both take O(log n), whereas a sorted list would make every insertion O(n).

Ranking purely by severity has a serious flaw. A minor case could be overtaken again and again by a steady stream of serious ones, and so wait indefinitely. This is called starvation. To avoid it we use

*priority(r, t) = P[severity(r)] + a × (t − arrival(r))*

where P turns severity levels 1 to 5 into 10, 20, 40, 70 and 100 points, and the aging rate a is 1 point per minute. The points rise faster than linearly because we judged the clinical gap between severity 4 and 5 to be larger than the gap between 1 and 2. With a = 1, a severity-1 case that has been waiting overtakes a freshly arrived severity-5 case after 90 minutes. That sets an upper limit on how long anyone can be left waiting.

Since priorities change as time passes, you might expect the heap to need re-sorting every minute. It does not, and this was the part of the design we found most satisfying. Expanding the formula gives

*priority = (P − a × arrival) + a × t*

The last term, a × t, is the same for every request at any given moment, so it never changes who ranks above whom. The heap therefore stores the fixed key P − a × arrival once, and the order stays correct as time moves on. This only works because the aging is linear; a non-linear rule would force us to rebuild the heap. We tested the idea by comparing the heap's winner with a brute-force recalculation of every priority, over 200 random trials. They never disagreed.

For comparison we also built two baseline policies: first-come-first-served (FIFO) and severity-only.

## 4. Ambulance selection

The obvious way to choose an ambulance is to compute a route from every free ambulance to the patient and keep the shortest. With k free ambulances, that means k shortest-path searches.

We did it differently. Roads are two-way, so the travel time from ambulance to patient equals the travel time from patient to ambulance. That lets us run a single Dijkstra search starting at the patient and stop the moment it settles a junction where a free ambulance is waiting. Dijkstra settles junctions in order of increasing travel time, so the first ambulance it finds is guaranteed to be the closest one by road. The same routine, pointed at hospital junctions instead, gives us the nearest hospital.

We also implemented the "straight-line" method, which is what a person glancing at a map would probably do, so we could measure how often it picks the wrong vehicle. The notebook contains all three methods (single search, k searches and straight-line) so their results can be compared side by side.

## 5. Route optimisation: Dijkstra and A\*

We implemented two routing algorithms and compared them.

**Dijkstra's algorithm** keeps a min-heap of frontier junctions ordered by travel time from the start, and repeatedly settles the nearest one. It knows nothing about where the destination is, so it spreads outward in every direction like a ripple on water. With a binary heap it runs in O((V + E) log V) time, and it is optimal whenever weights are non-negative.

**A\*** orders its heap by f(n) = g(n) + h(n), where g is the time already spent getting to the junction and h is an estimate of the time still to go. Our estimate is the straight-line distance to the destination divided by the top speed of 40 km/h. It can never be too high, because every road is at least as long as the straight line and no road is faster than 40 km/h. A heuristic with this property is called admissible, and it guarantees that A\* returns an optimal route. Our heuristic is also consistent, so each junction is settled only once. A\* has the same worst-case complexity as Dijkstra, but it is pulled toward the destination and usually expands far fewer junctions.

We checked both algorithms in two stages. First we ran them on a four-node graph small enough to solve by hand (the answer is a cost of 4 via A, B, C, D), including a case where the target cannot be reached. Then we compared them against each other on 300 random junction pairs in the Arua network. The costs matched exactly every time.

## 6. Performance evaluation

### 6.1 Method

We looked at three things: execution time, computational complexity and solution quality. Time was measured with a high-resolution timer and averaged over 40 random queries for each city size. Seconds depend on the machine, so we also counted the junctions each algorithm expanded, which measures the work done regardless of hardware. To see how things scale, we generated synthetic cities from 98 to 6,400 junctions with the same generator. All random seeds are fixed.

### 6.2 Routing: Dijkstra versus A\*

| Junctions | Roads | Dijkstra (ms) | A\* (ms) | Dijkstra expanded | A\* expanded | Ratio |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 98 | 171 | 0.034 | 0.031 | 47 | 26 | 1.8 |
| 400 | 723 | 0.150 | 0.124 | 198 | 104 | 1.9 |
| 900 | 1,652 | 0.291 | 0.208 | 367 | 167 | 2.2 |
| 2,025 | 3,795 | 0.994 | 0.831 | 1,108 | 597 | 1.9 |
| 3,600 | 6,750 | 1.892 | 1.385 | 1,803 | 901 | 2.0 |
| 6,400 | 11,990 | 3.304 | 2.627 | 3,048 | 1,663 | 1.8 |

*Table 1. Average per-query performance. Ratio = Dijkstra expansions divided by A\* expansions.*

Across every query, the largest difference in route cost between the two algorithms was exactly zero, so A\* gave up nothing in quality. It expanded roughly half as many junctions as Dijkstra at every size, and it was 10 to 30% faster in wall-clock time. The time saving is smaller than the saving in expansions because A\* has to compute its heuristic at every node, and that costs something.

We also fitted a line to the log-log plot of time against city size. The growth exponent came out at about 1.12 for Dijkstra and 1.08 for A\*. That fits the theory: with E proportional to V, a purely linear algorithm would give 1.0, and the log factor nudges it slightly higher.

In practice, on a road network of a few thousand junctions, both algorithms answer in a few milliseconds, much faster than a dispatcher can speak. A\* would matter more for far larger networks, or for a system handling many requests at once.

### 6.3 Ambulance selection

| Free ambulances (k) | Single search (ms) | k searches (ms) | Expanded (single / k) | Speed-up |
| ---: | ---: | ---: | :---: | ---: |
| 2 | 0.44 | 1.33 | 495 / 1,545 | 3× |
| 5 | 0.19 | 3.19 | 207 / 3,820 | 17× |
| 10 | 0.13 | 6.63 | 142 / 7,883 | 49× |
| 20 | 0.08 | 13.35 | 77 / 15,753 | 168× |
| 40 | 0.05 | 26.65 | 39 / 31,613 | 527× |

*Table 2. Ambulance selection on a 1,600-junction city (30 random incidents per row).*

Both methods returned identical travel times in every trial, so the single search is a pure efficiency gain. What surprised us is that its cost actually falls as k grows. With more ambulances around, the search runs into one sooner and stops earlier, while the naive method simply does k times the work. If there is one idea in this project we would point to, it is this one: searching from the patient instead of from the ambulances turns k problems into one.

We also looked at quality. In 400 random incidents with six free ambulances, we compared the truly nearest ambulance by road with the one that was nearest in a straight line. They disagreed in 53 cases (13.2%). When they did, the straight-line choice delayed the patient by 1.20 minutes on average and by as much as 3.81 minutes. Our city is grid-like, so in a place with more rivers, dead ends or one-way streets we would expect the gap to be bigger.

### 6.4 Prioritisation quality

To compare the three queueing policies we ran a discrete-event simulation of 30 independent four-hour days per scenario. In the model, an ambulance is busy for its drive to the scene, 5 minutes on scene, the drive to the nearest hospital, and 10 minutes of handover. After that it waits at the hospital where it dropped the patient. Response time is the time spent waiting in the queue plus the drive to the scene.

Table 3 shows mean response times in minutes for the stressed scenario of 70 calls and six ambulances. The busier but less extreme scenario (60 calls, eight ambulances) is in the notebook.

| Policy | Severity 5 | Severity 4 | Severity 1-2 | Worst response | Severity-weighted |
| --- | ---: | ---: | ---: | ---: | ---: |
| FIFO | 45.2 | 46.2 | 47.2 | 150.1 | 46.5 |
| Severity only | 12.5 | 14.5 | 88.3 | 267.0 | 25.8 |
| Severity + aging (proposed) | 12.6 | 24.1 | 67.9 | 151.0 | 33.9 |

*Table 3. Mean response time in minutes, stressed scenario (70 calls, 6 ambulances, 30 simulated days).*

Three things stood out.

**Prioritisation moves the wait around; it does not remove it.** The overall mean response time was almost the same under all three policies, about 46 minutes. Reordering a queue does not create extra ambulances. What changes is who ends up waiting. FIFO makes a critical patient wait as long as someone with a minor problem.

**Severity-only dispatch helps the critical patients and hurts everyone else.** It brings the severity-5 wait down from 45.2 to 12.5 minutes, but low-severity patients wait 88 minutes on average and up to 267 minutes in the worst case. This is the starvation problem showing up in real numbers.

**Aging fixes most of that.** Severity-5 patients still wait about the same (12.6 minutes), but the worst-case response drops from 267 to 151 minutes, which matches FIFO's worst case, and the low-severity average falls from 88 to 68 minutes. The price is that severity-4 patients wait longer than under severity-only dispatch (24.1 against 14.5 minutes). That is a real trade-off, and it is for a medical authority to judge, not for us.

When the system is less loaded, as in the 60-call scenario, the three policies differ much less. Even there, though, severity-only dispatch produced a worst response of 136 minutes, so starvation can happen even when things are not that busy.

We also tested how sensitive the results are to the aging rate. At a = 0 the worst response is 267 minutes. At a = 1 it falls to 151 minutes, while severity-5 patients still wait only 12.6 minutes. Raising a to 2 and 4 pushes the severity-5 wait up to 19.4 and 29.3 minutes in return for only modest further gains in the worst case. So a = 1 looks like a sensible knee in the trade-off curve. Even so, the final value is a policy decision, and clinicians should be the ones to set it.

## 7. Complexity summary

| Operation | Algorithm | Time (space) |
| --- | --- | --- |
| Enqueue / dequeue emergency | Binary heap, static aging key | O(log n) (O(n)) |
| Point-to-point route | Dijkstra | O((V+E) log V) (O(V)) |
| Point-to-point route | A\* with admissible heuristic | O((V+E) log V) worst case, fewer expansions in practice (O(V)) |
| Ambulance selection | One early-stopping Dijkstra | O((V+E) log V) (O(V)) |
| Naive ambulance selection | k Dijkstra runs | O(k (V+E) log V) (O(V)) |

*Table 4. Theoretical complexity of each component.*

We also benchmarked the heap on its own. The time divided by n log n stayed at about 0.2 microseconds from 1,000 up to 256,000 requests, which is what we would expect if n insertions and removals really do cost O(n log n).

## 8. Limitations and future work

The prototype rests on some simplifying assumptions that a real deployment would have to drop.

- **Traffic is a fixed snapshot.** Real travel times change through the day, so we would need weights that vary with time.
- **Ambulances stay where they finish.** After a drop-off, an ambulance waits at the hospital instead of going back to a station chosen to give the best coverage. Each ambulance also carries only one patient at a time.
- **Hospitals are all alike.** We do not model capacity or specialities such as trauma or maternity units, so the nearest hospital is not always the right one.
- **The data is simulated, from a single city layout.** The absolute response times should not be read as predictions for Arua. The comparisons between algorithms and between policies are the meaningful results.

For future work we would like to import the real road network from OpenStreetMap, add hospital constraints, reposition idle ambulances ahead of demand, speed up routing with landmark or hierarchical methods, and check the severity weights with clinicians.

## 9. Conclusion

We built a complete dispatch pipeline out of a few classical algorithms, and tested each part against the alternatives.

- A heap with a static aging key ranks calls in O(log n) time. It protects critical patients and still stops anyone from being starved.
- Searching outward from the patient finds the road-nearest ambulance in one pass. At a fleet of forty, this was up to hundreds of times faster than the naive method, and the answer was never worse.
- Dijkstra and A\* found identical optimal routes, with A\* expanding about half as many junctions.
- Straight-line reasoning picked the wrong ambulance in 13% of dispatches.
- The choice of priority policy changes who gets served first, not the average wait. That is why it should be made openly, by the people responsible for the service.

## References

Dijkstra, E. W. (1959). A note on two problems in connexion with graphs. *Numerische Mathematik*, 1, 269-271.

Hart, P. E., Nilsson, N. J. and Raphael, B. (1968). A formal basis for the heuristic determination of minimum cost paths. *IEEE Transactions on Systems Science and Cybernetics*, 4(2), 100-107.

Cormen, T. H., Leiserson, C. E., Rivest, R. L. and Stein, C. (2022). *Introduction to Algorithms*, 4th ed. MIT Press.

## Contribution statement

Complete this table truthfully before submission, replacing the bracketed placeholders. Each member should be able to explain and defend the part they are credited with. The roles below are a suggested split for a group of five; for three or four members, merge adjacent roles.

| Member | Responsibilities Assigned|
| --- | --- |
| [Name 1] | Pipeline design, simulation loop, notebook integration, final testing |
| [Name 2] | Dataset generation, CSV schema, graph data structure, data description |
| [Name 3] | Heap-based queue, aging formula, policy comparison, sensitivity analysis |
| [Name 4] | Dijkstra, A\*, ambulance selection, hand-verified tests |
| [Name 5] | Benchmarks, complexity analysis, charts, report editing |

