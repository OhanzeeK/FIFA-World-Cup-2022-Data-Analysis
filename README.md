# FIFA World Cup 2022 — Player Performance Analysis

**Author:** Ohanzee Karneh**Tools:** PySpark (data cleaning & transformation) | Power BI (visualization & dashboarding)**Data Source:** https://www.kaggle.com/datasets/swaptr/fifa-world-cup-2022-player-data/code
---

## Project Overview

This project explores player and team performance at the 2022 FIFA World Cup in Qatar using two datasets from Kaggle covering general player statistics and defensive player statistics. The data was cleaned and merged using PySpark, then visualized in Power BI to explore scoring efficiency, team discipline, defensive intensity, and expected goal performance.

---

## Data Cleaning Pipeline (PySpark)

1. **Column Selection** — Reduced from 53 combined columns to 20 relevant fields
2. **Dataset Join** — Merged general stats and defensive stats on `player` and `position` (inner join)
3. **Zero-Minute Filter** — Removed players who did not play (0 minutes) to prevent skewing averages
4. **Duplicate Removal** — Dropped any duplicate rows created during the join
5. **Null Handling** — Filled null xG-related columns with 0 (no shot activity = 0 expected goals); filled null `club` values with "N/A"
6. **Schema Validation** — Verified all column types were correctly inferred

---

## Questions & Findings

### Q1: Which team has the most rough play?

**Methodology:** Created a Card Penalty Index based on FIFA's official Fair Play scoring system. Under FIFA's tiebreaker rules, disciplinary deductions are calculated as:

| Card Type | Points |
| --- | --- |
| Yellow card | -1 |
| Indirect red card (second yellow) | -3 |
| Direct red card | -4 |
| Yellow + direct red (same game) | -5 |

For this analysis, a simplified version was used:

- **Yellow cards = 1 point**
- **Red cards = 3 points**

The data source does not discriminate between indirect and direct red cards, so indirect cards were assumed.

**Formula (DAX):**

```dax
Card Penalty Index = SUM('cupData'[cards_yellow]) + SUM('cupData'[cards_red]) * 3

```

**Key Finding:** Argentina accumulated the highest Card Penalty Index (17), driven entirely by yellow cards (17 yellows, 0 reds). Netherlands (15) was notable for combining both card types (1 red, 12 yellows). Only 4 red cards were issued across the entire tournament.

**Source:** FIFA Fair Play point deduction system as described in FIFA's World Cup regulations. Referenced from:

- NBC Sports (2022). "FIFA's Fair Play Rule and How It Affects Points in World Cup Matches." [https://www.nbcsports.com/boston/world-cup-2022/fifas-fair-play-rule-and-how-it-affects-points-world-cup-matches](https://www.nbcsports.com/boston/world-cup-2022/fifas-fair-play-rule-and-how-it-affects-points-world-cup-matches)

---

### Q2.1: Who were the top scorers in the tournament?

**Visual:** Clustered bar chart (horizontal) — Top 10 players by total goals.

**Key Finding:** Kylian Mbappé led with 8 goals, followed by Lionel Messi with 7. A notable drop-off occurs after the top 2, with the remaining 8 players tied between 3-4 goals each.

---

### Q2.2: Who had the best goals/minutes played ratios?

**Methodology:** Used the pre-calculated `goals_per90` field. Players with fewer than a match's worth of playtime were excluded to prevent super-subs/finishers from skewing the data.

**Visual:** Scatter plot (X = minutes played, Y = goals, bubble size = goals_per90) paired with a detailed table.

**Key Finding:** Marcus Rashford had the highest goals/90 rate (1.93) among top scorers, scoring 3 goals in just 140 minutes. Gonçalo Ramos (1.78) was similarly efficient. By contrast, high-volume scorers like Messi (0.91) and Mbappé (1.20) played significantly more minutes.

---

### Q3: Which countries have the best defensive intensity?

**Methodology:** Built a weighted Defensive Intensity Index based on football analytics literature, normalized per 90 minutes and averaged per player to ensure fair comparison regardless of tournament progression.

**Weights:**

| Statistic | Weight | Justification |
| --- | --- | --- |
| Interceptions | 30% | Highest-valued — indicates game intelligence and anticipation; cleanest form of possession recovery |
| Tackles Won | 22% | Direct ball-winning action; ranked below interceptions because high tackle can indicate poor positing and lead to cards |
| Clearances | 18% | Remove immediate goal-scoring threats but do not regain poseession |
| Blocked Shots | 18% | Directly prevents goal-scoring opportunities but could be avoided with better play upfield|
| Blocked Passes | 12% | Disrupts buildup play but less directly impactful than shot blocks |

**Formula (DAX):**

```dax
DefenseScore = 
AVERAGEX(
    FILTER('cupData', [minutes_90s] > 0),
    ([interceptions] * 0.30 + [tackles_won] * 0.22 + [clearances] * 0.18 + [blocked_shots] * 0.18 + [blocked_passes] * 0.12)
    / [minutes_90s]
)

```

**Normalization:** Divided by `minutes_90s` to produce a per-90-minute rate, preventing teams with more matches (deeper tournament runs) from naturally accumulating higher totals. `AVERAGEX` was used instead of `SUMX` to prevent teams with larger squads from scoring higher.

**Key Finding:** Ghana (1.40), Tunisia (1.25), and Saudi Arabia (1.20) led in defensive intensity. Tournament average was 0.89. Notably, teams eliminated earlier had higher defense scores, meaning this metric could more accurately reflect the teams most pressured by opposing possession.

**Source:** https://bleacherreport.com/articles/1722602-which-stats-are-most-important-for-measuring-defenders


---

### Q4.1: How are the total tournament goals split across the teams?

**Visual:** Pie chart with the top 10 scoring countries individually and remaining teams grouped as "Other."

**Key Finding:** France (9.41%) and Argentina (8.82%) contributed the largest individual shares. However, "Other" (the remaining 22 teams combined) accounted for 39.41% of all goals — showing that goal-scoring was fairly distributed across the tournament with only a few standout teams

---

### Q4.2: How do the expected goals differ from the actual results?

**Methodology:** Compared total actual goals vs. total Expected Goals (xG) per team using a line and clustered column chart. xG is a statistical measure provided by Opta estimating the probability of a shot resulting in a goal based on factors like shot distance, angle, and type.

**Visual:** Combo chart — bars represent actual goals, line represents expected goals. Teams above the line over-performed; teams below under-performed.

**Key Finding:** The tournament as a whole slightly under-performed (170 actual goals vs. 174 expected goals). Teams like France significantly over-performed their xG (clinical finishing), while others under-performed despite creating high-quality chances.

---

## Limitations

1. **Defensive metrics are contextual** — High defensive action rates can indicate a team under pressure rather than defensive excellence. Positional defending (forcing opponents into poor decisions) is not captured by event-based statistics.
2. **Card Penalty Index simplification** — The dataset does not distinguish between direct red cards (-4 in FIFA's system) and indirect reds (-3). All reds were treated as -3.
3. **Goals Per 90 threshold** — The 90-minute minimum excludes some impactful substitute appearances but is necessary to prevent statistical noise.
4. **xG model uncertainty** — Expected Goals is a model estimate. The sourced Kaggle dataset uses Opta's formula, which may cause differences if this study was repeated using another system.

---

## Tools & Technologies

| Tool | Purpose |
| --- | --- |
| **PySpark** | Data ingestion, cleaning, transformation, joining datasets |
| **Jupyter Notebook** | Development environment for PySpark pipeline |
| **Power BI** | Interactive dashboard design, DAX measures, data visualization |
| **Power Query** | Additional data transformations (Team Display grouping, column renaming) |

---

## Repository Contents

| File | Description |
| --- | --- |
| `WorldCup2022Cleaning.ipynb` | PySpark cleaning and transformation notebook |
| `world_cup_2022_cleaned.csv` | Final cleaned dataset exported for Power BI |
| `player_stats.csv` | Raw general player statistics dataset |
| `player_defense.csv` | Raw player defensive statistics dataset | 
| `Football Analysis.pbix` | Power BI dashboard file |
| `images/` | Dashboard screenshots |
| `README.md` | Project overview and documentation |

