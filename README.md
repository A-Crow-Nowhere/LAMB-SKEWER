# LAMB-SKEWER
Long-read Allocation Model Builder and Structural Karyotype Estimation, Weighting, and Event Resolver


'                    ┌───────────────┐
BAM directory ─────▶│     LAMB      │
                    └───────┬───────┘
                            │
                allocation model + manifest
                    optional normalized BAMs
                            │
                            ▼
                    ┌───────────────┐
Original BAMs ─────▶│    SKEWER     │
                    └───────┬───────┘
                            │
                         CNV calls
                            +
                      per-read evidence
                            +
                  population/skew analysis
 '
