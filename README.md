## Ambulance Dispatch and Routing for Arua City: A Simulation Study of Priority Queuing, Dijkstra and A*

An algorithmic decision-support prototype for ambulance dispatch in Arua City, Uganda. It was built as a group project for the Algorithms course at Muni University.

When an emergency call comes in, a dispatcher has to decide which call to serve first, which ambulance to send, and which road to take. This project treats those three decisions as one pipeline and tests each step against simpler alternatives.

```
Emergency arrives -> PRIORITISE -> SELECT ambulance -> ROUTE to patient & hospital -> EVALUATE
                    (heap + aging)  (early-stopping Dijkstra)   (Dijkstra vs A*)      (time, complexity, quality)
```

## What is in this repository

| File | What it is |
| --- | --- |
| `Arua_Ambulance_Dispatch_Notebook.ipynb` | The notebook. It holds all the code, the experiments and short explanations of each concept. |
| `Arua_Ambulance_Dispatch_Report.pdf` | The full written report (Word version). |
| `Arua_Ambulance_Dispatch_Report.md` | The same report in Markdown. |
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
jupyter Arua_Ambulance_Dispatch_Notebook.ipynb
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
| 7 | Complexity summary and limitations 