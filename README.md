# Temporal Community Detection in Stock Market Networks

A network science project (Leiden University) that studies how the community structure of the **NASDAQ-100** changes over time. A *community* is a group of stocks whose prices tend to move together.

The project builds stock correlation networks over rolling time windows and detects communities in each one with six methods: **Louvain**, **signed Louvain**, **Leiden**, **spectral clustering**, and temporal versions of Louvain and Leiden. It then compares the methods on modularity, stability over time, agreement with each other, conductance and within-community correlation.

Everything is in one notebook: [`FINAL_SEVERAL_METHODS.ipynb`](FINAL_SEVERAL_METHODS.ipynb).

## How it works

1. **Data.** The index members and their GICS sectors are read from Wikipedia, and daily closing prices (adjusted for splits and dividends) come from Yahoo Finance. The default is a fixed revision of the *Nasdaq-100* article from 21 Dec 2022, for two reasons: the current article no longer has the members table, and using the members at the start of the period avoids look-ahead bias. Stocks with missing prices are dropped and listed.
2. **Rolling correlations.** Daily returns are cut into windows (default: 14 days, moving 14 days each time). Each window gets its own Pearson correlation matrix.
3. **Networks.** Each correlation matrix becomes a sparse network with a fixed budget of `3n − 6` edges, the edge count of a planar maximally filtered graph:
   - Each stock gets a target degree proportional to its total similarity.
   - Edges are added from the most to the least similar pair while both stocks still have free degree.
   - The graph type sets the edges and weights:

     | Graph type | Edges and weights |
     |---|---|
     | `"w"` | weighted, positive correlations only |
     | `"n"` | the same edges as `"w"`, unweighted |
     | `"sw"` | signed and weighted: edges ranked by absolute correlation, so negative edges are kept |

4. **Community detection** in every window:

   | Method | Description |
   |---|---|
   | `louvain` | Louvain (`python-louvain`) on the positive edges |
   | `louvain_signed` | Our own Louvain for signed networks. It maximises `Q⁺ − Q⁻`, which rewards positive edges inside communities and negative edges between them. |
   | `leiden` | Leiden (`leidenalg`) on the positive edges |
   | `spectral` | Spectral clustering of the window's positive correlations into `N_COMMUNITIES` groups. It uses the full correlation matrix, because the sparse network is often disconnected. |
   | `louvain_temporal` | Signed Louvain plus `smoothing ×` the modularity of the partition on the previous window's communities, which rewards keeping earlier groups together |
   | `leiden_temporal` | Leiden started from the previous window's communities |

5. **Evaluation.** Each method gets a table with one row per window, and each table can be plotted:

   | Metric | Meaning |
   |---|---|
   | Modularity, number of communities | Strength and size of the community structure. Modularity is computed on the positive edges for every method. |
   | VI / NMI vs the previous window | Stability: how much a method's communities change over time (VI 0 = identical, NMI 1 = identical) |
   | VI / NMI between methods | Agreement between two methods in the same window |
   | Conductance | Share of a community's edge weight that leaves it (lower is better) |
   | Within-community correlation | Mean correlation of all stock pairs inside a community, from the window's full correlation matrix |

The notebook also builds one network over the **whole period**. It plots it coloured by sector and by community, prints every community with its main sector and members, and shows the communities of the last window.

## Requirements

Python 3.11 and:

```
pip install pandas numpy lxml networkx matplotlib scikit-learn python-louvain python-igraph leidenalg yfinance jupyter
```

Tested with pandas 2.1, numpy 1.26, networkx 3.2, matplotlib 3.8, scikit-learn 1.7, python-igraph 1.0, leidenalg 0.11 and yfinance 0.2.66. An internet connection is needed for Wikipedia and Yahoo Finance.

## Usage

1. Open `FINAL_SEVERAL_METHODS.ipynb` in Jupyter or VS Code.
2. Change the **Settings** cell if needed:

   | Setting | Default | Meaning |
   |---|---|---|
   | `WIKIPEDIA_URL` | Nasdaq-100 revision of 21 Dec 2022 | Page with the members table |
   | `START_DATE`, `END_DATE` | 2023-01-01, 2025-01-01 | Price history |
   | `MAX_TICKERS` | `None` | Use only the first N stocks (quick tests) |
   | `WINDOW_SIZE`, `STEP_SIZE` | `"14D"`, `"14D"` | Window length and step |
   | `MIN_PERIODS` | `5` | Skip windows with fewer trading days |
   | `GRAPH_TYPE` | `"w"` | `"w"`, `"n"` or `"sw"` |
   | `MODELS` | all six | Methods to run |
   | `N_COMMUNITIES` | `3` | Clusters for spectral clustering |
   | `TEMPORAL_SMOOTHING` | `0.5` | Weight of the temporal term in `louvain_temporal` |
   | `SEED` | `42` | Makes every run reproducible |
   | `CROSS_PAIRS` | 5 pairs | Method pairs compared in the agreement plots |

3. Run all cells.

To use the code from your own cell:

```python
market = MarketAnalyzer(WIKIPEDIA_URL, "2023-01-01", "2025-01-01", name="NASDAQ-100")
market.load_data()
market.main_temporal_pipeline(models=("leiden", "louvain_temporal"), window_size="30D", graph_type="sw")
market.plot_metric("nmi_prev")                     # any key of MarketAnalyzer.PLOTS
market.compute_conductance_over_time()["leiden"]   # one DataFrame per method
market.print_community_summary("leiden", window=-1)
```

## Code structure

| Notebook section | Contents |
|---|---|
| 1. Data | `load_constituents`, `load_prices`, `rolling_correlations` |
| 2. Networks | `cascade_rounding`, `build_pd_graph`, `positive_part`, `analyze_graph` |
| 3. Community detection | `louvain`, `leiden`, `spectral`, `signed_louvain`, `detect_communities` |
| 4. Metrics | `modularity`, `variation_of_information`, `normalized_mutual_information`, `conductance`, `average_within_correlation`, `community_sizes` |
| 5. Plotting | `draw_network`, `plot_over_time` |
| 6. `MarketAnalyzer` | Keeps the data, windows, graphs and partitions, and provides the tables, plots and summaries |
| 7–10 | Running the analysis and the results |

## Notes and limitations

- With 14-day windows each correlation matrix comes from about 10 trading days for about 95 stocks, so the estimates are noisy. The notebook prints a reminder. Longer windows (for example `"60D"`) give more stable networks.
- Louvain, Leiden and spectral clustering cannot handle negative weights. On `"sw"` graphs they use only the positive edges; the signed methods use all of them.
- Sectors are used for colouring and for the community summaries; they are not used as a ground truth.
- Yahoo Finance no longer has prices for some former members (for example ATVI, SGEN, WBA, EA and WBD), so these are dropped. The notebook lists every dropped ticker.
