# Beyond the Scoreline — Data Engineering & Validation Log

**Project:** Three iconic 2017/18 European football matches, prepared for Tableau  
**Source:** Public Wyscout event dataset, distributed through Figshare and referenced by the Wyscout public dataset repository  
**Stack:** Python, pandas, `requests`, `zipfile`, `json`, VS Code/Jupyter, Git/GitHub, CSV, Tableau (downstream)  
**Data grain:** One row per original Wyscout event  
**Status:** Three-match CSV exported and core integrity checks passed. A Unicode-name correction cell was provided; its execution and successful re-export have **not yet been confirmed** in the conversation.

## 1. Purpose and scope

The objective was to build a compact, carefully curated, event-level football dataset suitable for visual storytelling and spatial analysis in Tableau. Rather than load an entire league, the project selected **three notable 2017/18 matches** with contrasting narratives. Python was used **only for ingestion, transformation, quality assurance, and export**; football analysis and visualization are reserved for Tableau.

The intended Tableau questions include:

- Who progressed the ball effectively rather than simply accumulating completed passes?
- Where did teams and players conduct actions on the pitch?
- How did patterns change across halves, particularly during Schalke's 4–0 to 4–4 comeback?
- How do attacking volume, spatial position, and scoring outcomes differ between matches?

These are **questions for subsequent analysis**, not conclusions established by the ETL.

## 2. Discovery and data acquisition

1. Investigated the public Wyscout football event data and its GitHub documentation. Initial candidate download links were unsuccessful (HTTP 404); these were not used as data sources.
2. Switched to the working Figshare file download endpoint: `https://ndownloader.figshare.com/files/14464685`.
3. Used `requests` to fetch the response, `io.BytesIO` to hold it in memory, and `zipfile.ZipFile` to inspect the archive.
4. Confirmed the archive contained:
   - `events_England.json`
   - `events_Italy.json`
   - `events_Germany.json`
   - `events_Spain.json`
   - `events_France.json`
   - `events_European_Championship.json`
   - `events_World_Cup.json`
5. Read individual league JSON files from the archive and converted their arrays of event objects into pandas DataFrames. No separate download was necessary for Germany because its file was already in the archive.
6. Downloaded the Wyscout player reference JSON from `https://ndownloader.figshare.com/files/15073721` and loaded it into `players_df` to enrich event records with player names.

The work was developed incrementally in a VS Code Jupyter notebook (`football_events.ipynb`) in the local project directory. Notebook changes were committed to GitHub; an earlier GitHub notebook-preview/“Invalid Notebook” issue was resolved during setup.

**Reproducibility note:** The notebook should remain the authoritative source for exact import statements, HTTP handling, player-name extraction, and the complete implementation of the transformation function. The Figshare links above identify the downloads used, but external hosting and availability can change.

## 3. Match selection and research questions

The selection was deliberate, not a random sample. The first two matches contrasted attacking intensity and a low-scoring outcome. The third match was selected through questions about preferred league, underdog narratives, player interest, ball progression, tactical contrasts, and dashboard ambition. Several candidates were discussed, including Barcelona fixtures, before settling on the Revierderby.

| Competition | Match | Date | Wyscout `matchId` | Events | Recorded goals |
|---|---|---|---:|---:|---:|
| Premier League | Liverpool 4–3 Manchester City | 2018-01-14 | 2499943 | 1,876 | 7 |
| Serie A | Napoli 0–1 Juventus | 2017-12-01 | 2576105 | 1,818 | 1 |
| Bundesliga | Borussia Dortmund 4–4 Schalke 04 | 2017-11-25 | 2516847 | 1,595 | 8 |
| **Total** | **3 matches** | | | **5,289** | **16** |

The third match was chosen because Schalke came back from 4–0 down, creating a compelling opportunity to compare event locations and ball movement between periods. No further matches were planned.

## 4. Raw event structure and initial profiling

The league event files used a consistent **12-column** raw structure:

```text
eventId, subEventName, tags, playerId, positions, matchId,
eventName, teamId, matchPeriod, eventSec, subEventId, id
```

Important nested fields:

- `positions`: list of coordinate objects (`x`, `y`), generally representing the start and sometimes end of an event.
- `tags`: list of tag objects with numeric `id` values; interpreted as event properties, not separate event records.
- `matchPeriod`: first or second half, paired with `eventSec` (seconds within that period).
- `id`: unique event identifier, preserved for traceability.

Checks performed during profiling included raw column structure, row count, distinct match IDs, event types, period counts, duplicate event IDs, goal-related tags, team IDs, and assigned/unassigned player IDs.

For Germany specifically, `events_Germany.json` loaded as **519,407 events across 306 matches**, with the same 12 columns and **zero duplicate event IDs**. Filtering to `matchId=2516847` returned **1,595 events**, **786 first-half** and **809 second-half**, and **zero duplicate IDs**.

## 5. Transformation design

A reusable function, `prepare_match_events(raw_df, metadata, team_map, players_df)`, was created and used to normalize individual matches into the same event schema. The function accepts the raw event DataFrame, match metadata, team-ID-to-name mapping, and player reference DataFrame.

Transformation steps:

1. **Filter by exact match ID** to prevent cross-match contamination.
2. **Attach match metadata:** competition, season, match date, and readable match name.
3. **Map team IDs to team names**, then validate the mapping for completeness.
4. **Join player metadata** by `playerId` to add readable names. Preserve events with `playerId=0` rather than dropping them; add `player_assigned` to distinguish unassigned events from failed lookups.
5. **Flatten coordinates** from nested `positions` arrays into `start_x`, `start_y`, `end_x`, and `end_y`. Preserve missing end positions as null rather than fabricating them.
6. **Standardize event timing** using `matchPeriod`, `eventSec`, `match_seconds`, `match_minute`, and `minute_label`.
7. **Flatten tag-derived indicators**, including accurate (`1801`), inaccurate (`1802`), goal (`101`), and assist (`301`) flags.
8. **Separate goals from goalkeeper concession records**: `is_goal_scored` applies to goal-tagged `Shot` events, while `is_goal_conceded_event` identifies goal-tagged `Save attempt` events. This avoids double-counting the same goal.
9. **Preserve original tags** in `tags_json` so that downstream definitions can be reviewed or expanded without losing the underlying information.
10. **Select and order a common schema** across matches before concatenation.

### Initial 31-column schema

```text
id, matchId, competition, season, match_date, match_name,
teamId, team_name, playerId, player_name, player_assigned,
eventId, subEventId, eventName, subEventName,
matchPeriod, eventSec, match_seconds, match_minute, minute_label,
start_x, start_y, end_x, end_y,
is_accurate, is_inaccurate, is_goal, is_goal_scored,
is_goal_conceded_event, is_assist, tags_json
```

### Three additional scoring columns (final 34-column schema)

- `is_own_goal`: whether the event carries Wyscout tag `102`.
- `is_scoring_event`: `is_goal_scored OR is_own_goal`.
- `scoring_team_name`: team credited with the goal; for own goals, the **opposing team**, not the event's team.

The original event's `teamId` and `team_name` are retained unchanged. Scoring attribution is a separate derived field.

## 6. Investigations, edge cases, and resolutions

### A. Goal-tag duplication

**Question:** Why were goal-tagged events more numerous than actual goals?  
**Test:** Group goal-tagged records (`tag 101`) by `eventName`.  
**Finding:** Goal-tagged `Save attempt` records represent the conceding goalkeeper's event and must not be counted as additional goals.  
**Resolution:** Count `Shot` + goal tag for regular goals; retain goalkeeper records as separate events.

### B. Dortmund–Schalke own goal

**Question:** Why did Dortmund–Schalke have seven goal-tagged `Shot` events but an official 4–4 result?  
**Test:** Inspect all 15 goal-tagged records (seven `Shot`, eight `Save attempt`) and examine events near the unmatched first-half goalkeeper event.

The decisive sequence:

| Half | `eventSec` | Team ID | Event | Tags / interpretation |
|---|---:|---:|---|---|
| 1H | 1064.515223 | 2447 | Free-kick cross | Preceding Dortmund delivery |
| 1H | 1066.069604 | 2449 | Touch | Tag `102` = own goal |
| 1H | 1069.521615 | 2449 | Save attempt | Tag `101` = goal conceded |

**Finding:** Schalke (`2449`) recorded an own-goal `Touch`, benefiting Dortmund (`2447`). The scorer-side event is not a normal goal-tagged shot.  
**Resolution:** Add `is_own_goal`, `is_scoring_event`, and `scoring_team_name`; reverse team attribution only for own goals.

A cross-match scan of tag `102` found **zero** own-goal events in Liverpool–Manchester City, **zero** in Napoli–Juventus, and **one** in Dortmund–Schalke. The resulting scoring-team totals were validated as **Borussia Dortmund 4, Schalke 04 4**.

### C. Missing player names

**Question:** Were blank player names failed metadata joins?  
**Test:** Compare null names against the `player_assigned` flag / `playerId`.  
**Finding:** Missing names occurred on unassigned events; **zero assigned-player records lacked names**.  
**Resolution:** Keep unassigned events and null player names; do not fabricate identities or delete events.

### D. Missing spatial endpoints

**Question:** Do all events have both a start and end position?  
**Test:** Count null coordinate fields by match and check coordinate ranges.  
**Finding:** Every event had a start coordinate. There was **one missing end-coordinate pair** in Napoli–Juventus and **one** in Dortmund–Schalke; Liverpool–Manchester City had none. Observed non-null coordinates were between **0 and 100**.  
**Resolution:** Retain null endpoints; do not interpret all event types as passes or movements.

### E. Escaped player names

**Question:** Why were some accented names displaying as literal Unicode sequences?  
**Test:** Search `player_name` for regex `\\u[0-9a-fA-F]{4}`.  
**Finding:** **929 event rows** contained unresolved Unicode escape sequences across **14 distinct names** (e.g., `S. Ag\\u00fcero`, `G. Higua\\u00edn`, `M. Pjani\\u0107`, `M. Ham\\u0161\\u00edk`, `M. G\\u00f6tze`).  
**Proposed resolution:** Replace each literal `\\uXXXX` sequence with the corresponding Unicode code point using a targeted `re.sub` callback; preserve existing correctly encoded characters. Re-export the CSV and verify zero unresolved sequences.  
**Status:** The correction cell was provided, but its successful execution has **not yet been reported**. This item remains a final check.

## 7. Validation record

### After combining the first two matches

- **3,694 total events** and **3,694 unique event IDs**.
- **Two** distinct matches.
- **Zero** duplicate IDs.
- Goals: Liverpool–Manchester City **7**; Napoli–Juventus **1**.
- **Zero** missing team names.
- **Zero** missing assigned-player names.

### Dortmund–Schalke transformed dataset

- **1,595 rows**.
- **34 columns**, identical in order and structure across all three transformed match DataFrames.
- **One** own goal.
- Scoring attribution: **4 goals per team**.
- **Zero** missing team names.

### Final three-match export

The notebook concatenated `master_events`, `napoli_master`, and `dortmund_master` with `pd.concat(..., ignore_index=True)` and ran assertions before export:

- Row count: **5,289**.
- Distinct matches: **3**.
- Unique event IDs: **5,289**; **zero duplicates**.
- Column count: **34**.
- Team names complete.
- Assigned-player names complete.
- Goal totals: **7**, **1**, and **8**, respectively.

The CSV was written with `index=False` and `encoding='utf-8-sig'`, then read back using pandas and checked again for row count, unique event IDs, and distinct match count.

**Confirmed export:** `data/processed/football_events_master.csv`  
**Reported file size:** approximately **1.36 MB**.

### Final audit results

| Match | Events | Missing start X/Y | Missing end X/Y | Missing player names* | Goals | Own goals |
|---|---:|---:|---:|---:|---:|---:|
| Dortmund 4–4 Schalke | 1,595 | 0 / 0 | 1 / 1 | 122 | 8 | 1 |
| Liverpool 4–3 Manchester City | 1,876 | 0 / 0 | 0 / 0 | 143 | 7 | 0 |
| Napoli 0–1 Juventus | 1,818 | 0 / 0 | 1 / 1 | 108 | 1 | 0 |
| **Total** | **5,289** | **0 / 0** | **2 / 2** | **373** | **16** | **1** |

*Missing player names correspond to unassigned events; there were **zero missing assigned-player names**. All records had non-null `match_seconds`. Observed coordinates were in the Wyscout 0–100 range.

## 8. Data model and Tableau considerations

- **Event grain:** One row per Wyscout event. `id` is the event key; `matchId` identifies the fixture.
- **Team attribution:** `team_name` describes the team associated with the original event. `scoring_team_name` describes who receives credit for a goal, including own goals.
- **Spatial interpretation:** Wyscout coordinates are normalized 0–100 and may be expressed relative to attacking direction. Do **not** assume they are fixed real-world stadium coordinates without checking the provider's conventions.
- **Missing endpoints:** A null `end_x`/`end_y` is not automatically a failed record or zero-distance action.
- **Timing:** `eventSec` is period-relative. The constructed `match_seconds` uses a standard second-half offset; stoppage time and chronological sequencing should be considered when designing minute-level visualizations. Keep `matchPeriod` and `eventSec` available.
- **Possession:** No validated possession-sequence ID was created in this phase. Event count is not possession count.
- **Ball progression:** Forward progression, progressive passes, and entries into dangerous areas require explicit, documented metric definitions. These have **not** yet been computed or validated.
- **Goals:** `is_goal` by itself is insufficient for counting goals, because goalkeeper events can also carry goal tag `101`. Use `is_scoring_event` and `scoring_team_name` for the validated matches.
- **Completeness:** The dataset is a **curated three-match sample**, not representative of all 2017/18 fixtures or all five leagues.

## 9. Reproduction checklist

1. Open the VS Code Jupyter notebook and confirm Python dependencies are available (`pandas`, `requests`, standard-library `io`, `zipfile`, `json`, `re`, `pathlib`).
2. Download the Figshare archive and player reference JSON; inspect archive members.
3. Load the England, Italy, and Germany event arrays as DataFrames.
4. Filter by exact match IDs `2499943`, `2576105`, `2516847`.
5. Apply the notebook's `prepare_match_events` function with each match's metadata and verified team map.
6. Derive own-goal and scoring-team fields consistently for all three matches.
7. Concatenate the three 34-column DataFrames.
8. Run the assertions for row counts, uniqueness, schema, assigned-player completeness, and scoreline totals.
9. Resolve literal Unicode escapes in player names, rerun the regex check, and confirm **zero** remaining escaped-name rows.
10. Export `data/processed/football_events_master.csv` with UTF-8 BOM and no pandas index; read it back and validate again.
11. Commit the notebook, this documentation, and the CSV according to repository policy.

## 10. Agent-assisted engineering workflow and attribution

This project followed a **human-directed, AI-agent-assisted data engineering workflow**. The project owner supplied the ground-truth requirements and verified observations: the selected fixtures, expected scorelines, source-file contents, event counts, schema outputs, anomalous goal records, and results of each validation cell. These observations were used as constraints for successive ETL decisions rather than assuming that generated code or inferred match details were correct.

The AI coding agent acted as an iterative engineering collaborator: it proposed extraction and transformation steps, helped define a reusable event schema, generated notebook cells, interpreted returned test outputs, identified discrepancies, and suggested targeted validation and corrections. The project owner executed the code locally in VS Code, inspected the results, provided corrections and verified facts, and approved the final export.

**Public-facing project description:**

> Developed a human-in-the-loop, agent-assisted ETL workflow to convert nested Wyscout football event JSON into a standardized, Tableau-ready dataset. Supplied match-level ground truths and validation results to guide iterative transformations, resolve own-goal attribution and player-name encoding issues, and verify data integrity across three historic fixtures.

**Scope of automation:** The agent assisted with code generation, reasoning, and validation design; local execution, source verification, match selection, and final acceptance remained human-controlled. This distinction preserves an accurate account of the workflow without implying an independently deployed autonomous pipeline.

## 11. Open items and boundaries

- **Confirm** execution of the Unicode correction and final CSV re-export; 929 escaped-name event rows were observed before the fix was proposed.
- If reproducibility outside the original notebook is important, consider a dependency manifest and an executable clean-run test later.
- Validate event-order assumptions and pitch-direction conventions before building progression metrics.
- Do not represent spatial or tactical hypotheses as established findings until analyzed in Tableau.

---

**Documentation basis:** Recorded notebook outputs and decisions shared during the project. Exact code is maintained in the notebook; this log describes the documented process and observed validations rather than claiming unobserved tests were performed.
