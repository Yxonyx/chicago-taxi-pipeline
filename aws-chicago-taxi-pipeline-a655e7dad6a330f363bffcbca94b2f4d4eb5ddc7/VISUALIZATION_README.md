```mermaid
flowchart TD
    subgraph S3["AWS S3 - Transformed Data"]
        T1["Taxi Trips CSVs"]
        W1["Weather CSVs"]
        C1["Company Master"]
        P1["Payment Type Master"]
        A1["Community Areas Master"]
        D1["Date Dimension"]
    end

    NB["Notebook 08_local_visualizations.ipynb"]

    T1 --> NB
    W1 --> NB
    C1 --> NB
    P1 --> NB
    A1 --> NB
    D1 --> NB

    NB --> F["Enriched Fact Table: trips_full"]
    F --> V["Local Visualizations"]
