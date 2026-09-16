# Netflix movies and TV shows: exploratory data analysis

Exploratory analysis of the Netflix Movies and TV Shows catalog (8,807 titles) from Kaggle. I dug into data quality issues, engineered a few features, and ran some statistical tests around what actually drives a title's content rating.

## Dataset

- Source: [Netflix Movies and TV Shows](https://www.kaggle.com/datasets/srisyra02/netflix-movies) (Kaggle)
- Size: 8,807 titles, 12 columns (`show_id`, `type`, `title`, `director`, `cast`, `country`, `date_added`, `release_year`, `rating`, `duration`, `listed_in`, `description`)
- Scope: the statistical section is filtered to `type == 'Movie'` (6,131 rows), so `rating` and `duration` mean the same thing across every row

## Data cleaning: what I actually found

### Missing `director` is structural, not random

3.1% missing for movies vs. 91.4% for TV shows. A show's director can change episode to episode, so the field gets left blank for series systematically, not because it's missing at random.

### Missing `cast` and `country` cluster by genre

Both fields are disproportionately missing for Stand-Up Comedy, Documentaries, and Children & Family content. Makes sense: a traditional "cast" or a single country of production doesn't really fit those formats.

### A column-shift bug, found and fixed

Three rows, all Louis C.K. stand-up specials, had duration values (`"74 min"`, `"84 min"`, `"66 min"`) sitting in the `rating` column, with `duration` itself blank. Looks like a source-side shift affecting titles with no assigned MPA/TV rating. I moved the value to the right column and set `rating` to a proper null instead of leaving a fake string in there.

### Missing `country` stays `NaN`

No reliable way to infer a film's country of origin from the other columns, so I left the missing values alone instead of guessing.

## Feature engineering

| Feature | Description |
|---|---|
| `year_added`, `month_added` | Parsed from `date_added` |
| `delay_years` | `year_added - release_year`: gap between release and Netflix availability |
| `duration_min` | Numeric minutes extracted from `duration` (movies only) |
| `primary_genre`, `primary_country` | First value from the comma-separated `listed_in` / `country` fields |
| `rating_ordinal` | `rating` mapped to an ordered numeric scale (`TV-Y` = 0 ... `TV-MA` = 11) for correlation analysis |

## Hypothesis testing: what drives a movie's rating?

I tested whether `rating` is associated with genre, country of origin, release year, and duration.

| Factor | Test | Result |
|---|---|---|
| Genre (`listed_in`, exploded) | Chi-square test of independence | p < 0.0001, significant |
| Genre vs. rating | Cramér's V | 0.260 |
| Country vs. rating | Cramér's V | 0.207 |
| Release year vs. rating | Spearman correlation | ρ = 0.164, p < 0.0001 |
| Duration vs. rating | Spearman correlation | ρ = 0.049, p = 0.0001 |

Genre has the strongest association with content rating out of everything tested, with country of origin close behind. Release year shows a weak but real trend toward stricter ratings over time. Duration is basically unrelated to rating: the p-value is small only because the sample is large, not because the effect means anything.

(These results cover the four factors I actually tested, not an exhaustive search of every possible driver.)

## Visualizations

- Top 15 genres by title count: `International Movies`, `Dramas`, and `Comedies` dominate. Genre tags aren't exclusive, so a title can land in several of these at once.
- Titles added per year: near zero through 2014, then explosive growth from 2015 to 2019 (peak around 1,400 titles in 2019), tapering off in 2020-2021. Read the 2021 drop with caution - the data collection looks incomplete for that year, so it's probably not a real slowdown.
- Rating distribution (mild to strict): the catalog skews mature, with `TV-MA` as the single largest rating.

## Repository structure

```
netflix-eda/
├── README.md
├── netflix.ipynb
├── netflix_titles.csv
└── requirements.txt
```

## How to run

```bash
pip install -r requirements.txt
jupyter notebook netflix.ipynb
```

Run all cells top to bottom. Nothing external to set up first.

## Stack

Python, pandas, numpy, matplotlib, scipy

## Limitations

- `country` and `listed_in` are multi-value fields. `primary_*` only keeps the first listed value, which is a simplification, not a full picture of co-productions or multi-genre titles.
- Cramér's V (categorical) and Spearman ρ (ordinal/numeric) aren't on a strictly identical scale, so comparing them directly tells you direction, not exact magnitude.
- The dataset looks like a snapshot from around late 2021, so recent-year figures probably undercount actual additions.
