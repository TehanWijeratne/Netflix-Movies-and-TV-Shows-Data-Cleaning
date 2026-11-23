# Netflix Movies and TV Shows: Data Cleaning

I cleaned the Netflix Movies and TV Shows dataset using Python and Pandas. The goal was not only to produce a tidy file, but to understand why the data was messy and to make clear, justified decisions about each problem. Every decision is documented here, including the issues I could not fully resolve.

## Project Purpose

This project applies a complete data-cleaning workflow to a real-world dataset. The data has the kinds of problems analysts meet in practice: missing values, dates stored as text, values entered in the wrong column, and cells with several values packed together.

## Objectives

- Look at the data properly before changing anything.
- Handle missing values column by column, based on what each gap means, instead of using `.dropna()`.
- Fix data types, especially `date_added`.
- Find values in the wrong column (`rating` and `duration`) and fix them only when the data backs it up.
- Clean the categorical columns (`country`, `listed_in`, `rating`).
- Split `duration` into a number and a unit.
- Check my work, compare it with the original, and export a cleaned CSV without touching the raw file.

## Dataset

- **Source:** [Netflix Movies and TV Shows on Kaggle](https://www.kaggle.com/datasets/shivamb/netflix-shows)
- **Licence:** CC0: Public Domain (shown on the Kaggle dataset page)
- **Size:** 8,807 rows and 12 columns (6,131 movies and 2,676 TV shows)
- **Raw file:** not included in this repository. Download it from Kaggle (see [How to Run](#how-to-run)). The cleaned file is included in `data/cleaned/`.

## Reference

I used Huong Ngo's article [How to Clean Your Data in Python](https://towardsdatascience.com/how-to-clean-your-data-in-python-8f178638b98d/) on Towards Data Science as a guide. I did not copy its row numbers or fixes. I checked every issue against my own copy of the dataset first.

## Tools

- Python 3.12
- Pandas 3.1.1 and NumPy 2.5.5
- Jupyter Notebook
- Git and GitHub

## Project Structure

```text
Netflix-Movies-and-TV-Shows-Data-Cleaning/
├── data/
│   ├── raw/                  # put netflix_titles.csv here (not tracked by Git)
│   └── cleaned/
│       └── netflix_titles_cleaned.csv
├── notebooks/
│   └── netflix_data_cleaning.ipynb
├── README.md
├── requirements.txt
└── .gitignore
```

## Methodology

I did all the cleaning on a copy of the data (`df_clean`) and kept the original (`df`) untouched, so I could compare the two at the end.

### 1. Looking at the data first

I used `.shape`, `.head()`, `.tail()`, `.columns`, `.info()`, `.dtypes`, `.describe()`, `.duplicated()` and `.value_counts()` before changing anything.

- 8,807 rows, 12 columns, and no duplicate rows. `show_id` is unique.
- `release_year` is the only numeric column. Everything else is text, including `date_added`.
- Three values in the `rating` column were actually durations (`74 min`, `84 min`, `66 min`).
- `cast`, `country` and `listed_in` hold several values in one cell, separated by commas.

### 2. Missing values

| Column | Missing | % | What I did |
|---|---:|---:|---|
| `director` | 2,634 | 29.91 | Filled with `"Unknown"` |
| `country` | 831 | 9.44 | Filled with `"Unknown"` |
| `cast` | 825 | 9.37 | Filled with `"Unknown"` |
| `date_added` | 10 | 0.11 | Left missing |
| `rating` | 4 | 0.05 | Left missing |
| `duration` | 3 | 0.03 | Recovered from `rating` (see section 3) |

What I found and why I chose this:

- **I did not use `.dropna()`.** It would have kept only 5,332 rows and left just 147 of the 2,676 TV shows. Any analysis after that would be wrong.
- **Most missing directors are TV shows.** 91.4% of TV shows have no director, compared with 3.07% of movies. A series usually does not have one credited director, so the gap is built into the data.
- **I chose `"Unknown"` as a placeholder** so I could keep every row and avoid blank cells in the exported file. I checked first that `"Unknown"` did not already exist in those columns. The downside is that it looks like real data, so it has to be left out when counting real directors, actors or countries.
- **I did not guess `date_added` from `release_year`.** Only 36.8% of titles were added in the year they were released, and the gap runs from -3 to 93 years. An estimate based on `release_year` would have been unreliable.

### 3. Data types and values in the wrong column

- **`date_added`:** 88 values started with a space, so I removed it with `.str.strip()`. Then I converted the column with `pd.to_datetime(..., format="%B %d, %Y")`. The 10 missing values became `NaT`. The dates run from 2008-01-01 to 2021-09-25.
- **Durations in the rating column:** three movies (`s5542`, `s5795`, `s5814`) had their duration in `rating` and nothing in `duration`. I moved the values into `duration` with `.loc` and set their `rating` to missing, because I don't know the real rating. That is why `rating` now has 7 missing values instead of 4.
- **`release_year`:** the type was already correct. But for 14 titles (12 TV shows and 2 movies), `date_added` is earlier than `release_year`. A likely explanation is that TV shows list the year of their latest season, but this is unproven. So I **flagged** these rows in a new column (`added_before_release`) and did not change them.

### 4. Categorical columns

- **`rating`:** I merged `UR` (Unrated) into `NR` (Not Rated) for 3 titles, because the data uses them the same way. I kept the old values in `rating_pre_merge`. This assumes the two labels don't need to be told apart for my purposes.
- **`country`:** splitting on commas gave 198 different strings, but after removing extra spaces only 124 were left, and 122 of those are real country names (the rest are an empty string and my `"Unknown"` placeholder). Seven entries had stray commas, like `", South Korea"`. I fixed them with a function and `.apply()` and saved the result in `country_clean`.
- **I did not merge any country names.** I found no spelling variants. Names like `West Germany`, `East Germany` and `Soviet Union` match the release years of their films, and they are not the same thing as Germany or Russia today, so I kept them. The column also includes territories such as Hong Kong and Puerto Rico.
- **`listed_in`:** the same extra-space problem made 42 genres look like 73. I made a cleaned column, `listed_in_clean`. `Movies` (57 titles) and `TV Shows` (16 titles) are not genres. They only repeat the `type` column, so I set them to missing in the cleaned column. That leaves 40 distinct genres.
- I kept the original `country` and `listed_in` columns.

### 5. Duration

`duration` mixes two formats: `90 min` for movies and `1 Season` or `2 Seasons` for TV shows. I added three columns:

| New column | What it holds |
|---|---|
| `duration_value` | The number as an integer |
| `duration_unit` | `min` (6,131 movies) or `Season` (2,676 TV shows). I merged `Seasons` into `Season` because the plural is only grammar |
| `duration_category` | `Short film (<40 min)`, `Standard (40-120 min)`, `Long (>120 min)`, `1 season`, `2-3 seasons`, `4+ seasons` |

My checks: every movie uses minutes, every TV show uses seasons, every `Season` (singular) is exactly 1, and every extracted number matches its original text. The category cut-offs are my own choice, not something in the data. I also looked at the extreme values (movies of 3 and 312 minutes). They are real: a short film and an interactive film.

### 6. Checking my work

- The data went from 8,807 x 12 to 8,807 x 19 (7 new columns).
- There are no duplicate rows and no duplicate title and type pairs.
- I compared every original column with the cleaned one. Changes appear only where I meant them to: `director` (2,634), `cast` (825), `country` (831), `rating` (6), `duration` (3).
- Remaining missing values: `date_added` 10, `rating` 7, `listed_in_clean` 73.
- I read the exported CSV back in to check its shape, types and missing counts, and confirmed the raw file did not change.

## Output

`data/cleaned/netflix_titles_cleaned.csv` has the 12 original columns (cleaned as described above) plus 7 new ones:

`added_before_release`, `rating_pre_merge`, `country_clean`, `listed_in_clean`, `duration_value`, `duration_unit`, `duration_category`

## How to Run

```powershell
git clone https://github.com/TehanWijeratne/Netflix-Movies-and-TV-Shows-Data-Cleaning.git
cd Netflix-Movies-and-TV-Shows-Data-Cleaning
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

1. Download `netflix_titles.csv` from the [Kaggle page](https://www.kaggle.com/datasets/shivamb/netflix-shows) and put it in `data/raw/`.
2. Run `jupyter notebook`, open `notebooks/netflix_data_cleaning.ipynb`, and choose **Kernel > Restart & Run All**.

On macOS or Linux, activate the environment with `source .venv/bin/activate`.

## Limitations and Remaining Issues

The cleaned dataset is not perfect. These issues remain open:

- 14 `release_year` values are earlier than `date_added`. I flagged them, but I can't tell which value is wrong.
- 7 ratings and 10 `date_added` values are still missing, and I found no reliable way to fill them in.
- `"Unknown"` in `director` covers two different cases: "not recorded" and "no single director".
- Seven country entries had stray commas. A country may have been lost before the comma, and I can't recover it.
- Historical states and territories are kept as listed, so `country` really means "countries and territories".
- The `Movies` type includes short films: 124 titles run under 30 minutes.
- 2021 is only a partial year (the data ends on 25 September 2021), so I can't compare its counts with earlier years.
- The `UR` to `NR` merge and the `duration_category` cut-offs are my decisions, not facts from the data.
- `"Unknown"` must be excluded when counting real directors, cast or countries.

## Key Takeaways

- Understand **why** a value is missing before deciding how to handle it. Deleting incomplete rows would have removed almost all TV shows.
- Correct a value only when the data provides evidence. Otherwise, flag it instead of changing it.
- Invisible problems, such as extra spaces, can silently double-count categories.
- Keep the original data and add new columns, so every change can be reversed and checked.

## Author

[TehanWijeratne](https://github.com/TehanWijeratne)

## Acknowledgements

Dataset by Shivam Bansal on Kaggle (CC0: Public Domain). Cleaning approach guided by Huong Ngo's article on Towards Data Science.
