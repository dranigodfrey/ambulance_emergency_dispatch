# # Arua City Emergency Dispatch & Routing: Improving Emergency Response Through Efficient Algorithms

An algorithmic decision-support prototype for ambulance dispatch in Arua City, Uganda. It was built as a group project for the Algorithms course at Muni University.

When an emergency call comes in, a dispatcher has to decide which call to serve first, which ambulance to send, and which road to take. This project treats those three decisions as one pipeline and tests each step against simpler alternatives.

```
Emergency arrives -> PRIORITISE -> SELECT ambulance -> ROUTE to patient & hospital -> EVALUATE
                    (heap + aging)  (early-stopping Dijkstra)   (Dijkstra vs A*)      (time, complexity, quality)
```

## What is in this repository

| File | What it is |
| --- | --- |
| `emergency_dispatch.ipynb` | The notebook. It holds all the code, the experiments and short explanations of each concept. |
| `Emergency_Dispatch_Report.docx` | The full written report (Word version). |
| `Emergency_Dispatch_Report.md` | The same report in Markdown. |
| `data/` | Created when you run the notebook (see below). Six CSV files describing the simulated city. |

## Getting started

### Requirements

- Python 3.8 or newer
- `pandas`, `numpy`, `matplotlib`
- Jupyter (Notebook, JupyterLab or VS Code with the Jupyter extension)

```bash
pip install pandas numpy matplotlib jupyter
```

The notebook otherwise uses only the standard library (`heapq`, `math`, `random`, `time`, `os`, `dataclasses`, `collections`).

### Running it

```bash
jupyter notebook emergency_dispatch.ipynb
```

Then choose **Kernel > Restart & Run All**. Run the cells in order, because later sections depend on objects created earlier (the graph, the dataset and the algorithm functions). The full run includes benchmarks on cities of up to 6,400 junctions and 30-day simulations, so expect it to take a little while.

The notebook creates a `data/` folder next to itself on the first run. Delete the folder to regenerate the dataset from scratch.

## The dataset

No public data on Arua's ambulance operations is available, so the dataset is simulated. **The coordinates are not real GPS points and the hospital names are placeholders.** If the authorities provide surveyed data, replace the CSV files in `data/` using the same columns. The algorithm code does not need to change.

| File | Contents |
| --- | --- |
| `nodes.csv` | Junctions with coordinates in km and a type (junction, hospital or station) |
| `roads.csv` | Road segments with length, speed, road class, congestion factor and travel time in minutes |
| `hospitals.csv` | Five hospitals with their junction and bed counts |
| `stations.csv` | Four ambulance stations |
| `ambulances.csv` | Eight ambulances, two per station |
| `emergencies.csv` | Sixty calls over four hours (Poisson arrivals), each with a severity from 1 to 5 and a category |

The road network is a jittered grid of 120 junctions and 209 roads, with faster arterial roads every fourth row and column, random congestion, and some local roads removed to mimic dead ends. Only the largest connected component is kept.

## Notebook outline

| Section | Topic |
| --- | --- |
| 0 | Setup and modelling assumptions (speeds, scene time, handover time) |
| 1 | Data representation: adjacency list graph, city generator, CSV dataset, map drawing |
| 2 | Prioritisation: binary heap with a severity and aging score, plus tests |
| 3 | Routing: Dijkstra and A\*, hand-verified tests and a visual comparison |
| 4 | Ambulance selection: one reversed search versus k searches versus straight-line |
| 5 | The dispatch simulation (discrete-event) and a traced example call |
| 6 | Performance evaluation: heap scaling, routing benchmarks, selection benchmarks, policy comparison, aging sensitivity |
| 7 | Complexity summary and limitations |

## Key design ideas

- **Adjacency list.** Road networks are sparse (about 3.5 roads per junction), so a dictionary of dictionaries needs V + E entries, where a matrix would need V².
- **Heap with a static aging key.** Priority is `severity points + aging x time waited`. Expanding this gives `(points - aging x arrival) + aging x now`, and the last term is the same for every request. The heap therefore stores `points - aging x arrival` once and never needs re-sorting. This relies on aging being linear.
- **Searching backwards to pick an ambulance.** Roads are two-way, so one Dijkstra search started at the patient, stopped at the first free ambulance it reaches, finds the road-nearest ambulance with a single search.
- **Admissible heuristic for A\*.** Straight-line distance divided by the top speed (40 km/h) never overestimates travel time, so A\* stays optimal.

### Main parameters

| Parameter | Value |
| --- | --- |
| Arterial / local speed | 40 / 25 km/h |
| Congestion multiplier | 1.0 to 1.5 per road |
| Time on scene | 5 min |
| Hospital handover | 10 min |
| Severity points (1 to 5) | 10, 20, 40, 70, 100 |
| Aging rate | 1 point per minute |

Each is a one-line change near the top of the notebook or in the relevant function.

## Summary of results

Figures below are taken from the project report and come from the fixed-seed experiments in the notebook.

- **Routing.** Dijkstra and A\* returned identical route costs on every query. A\* expanded about half as many junctions and was 10 to 30% faster. Both answer in a few milliseconds on cities of a few thousand junctions.
- **Ambulance selection.** The single reversed search gave the same travel times as the k-search method and was up to 527 times faster at 40 free ambulances.
- **Straight-line versus road distance.** Picking the ambulance nearest on the map was wrong in 13.2% of 400 trials, costing the patient 1.20 extra minutes on average and up to 3.81.
- **Prioritisation.** All policies had about the same mean response time (around 46 minutes in the stressed scenario). Severity-only dispatch cut severity-5 waits to 12.5 minutes but left low-severity patients waiting up to 267 minutes. Aging kept severity-5 waits at 12.6 minutes and brought the worst case down to 151 minutes.

## Reproducibility

Every random process uses a fixed seed (city generation, emergencies, benchmark queries and simulation days), so reruns should give the same dataset and results. Timing figures in milliseconds depend on your machine, which is why the notebook also reports the number of junctions expanded.

## Limitations

- Travel times are a static snapshot, with no time-of-day traffic.
- After a drop-off, an ambulance waits at the hospital instead of returning to a coverage-optimal station. Each ambulance carries one patient at a time.
- Hospital capacity and specialisation are not modelled, so the nearest hospital is not always the right one.
- The data is simulated from a single city layout. Compare algorithms and policies, but do not read the absolute response times as predictions for Arua.
- The severity points and aging rate are tunable policy choices and should be validated with clinicians.

## Possible extensions

- Import the real road network from OpenStreetMap (for example with `osmnx`).
- Add hospital bed availability and specialities.
- Relocate idle ambulances to coverage-optimal positions.
- Use time-dependent edge weights.
- Speed up routing with landmark (ALT) or contraction-hierarchy methods.

## Team

Group members: [names and registration numbers]

See the contribution statement in the report for each member's role.

## References

- Dijkstra, E. W. (1959). A note on two problems in connexion with graphs. *Numerische Mathematik*, 1, 269-271.
- Hart, P. E., Nilsson, N. J. and Raphael, B. (1968). A formal basis for the heuristic determination of minimum cost paths. *IEEE Transactions on Systems Science and Cybernetics*, 4(2), 100-107.
- Cormen, T. H., Leiserson, C. E., Rivest, R. L. and Stein, C. (2022). *Introduction to Algorithms*, 4th ed. MIT Press.