# Data Notes – Dare Careers Dashboard (DEM08)

## Source
Dataset provided by the instructor. Stored locally in `data/raw/` (not committed, see `.gitignore`).

```
data/raw/
├── Cloud Training/     (AWS Cloud track, 40 learners)
└── PowerBI Training/   (Power BI track, 100 learners)
    ├── Labs & Quizes/        Excel: sheets Labs, Quizzes
    ├── Participation/        Excel: one row per day
    ├── Status of Learners/   Excel: one row per learner
    └── Zoom Attendance/Week 1..10/  50 daily CSVs (05-Aug-2024 to 11-Oct-2024)
```

## Files
| File | Grain (one row =) | Layout | Key |
|---|---|---|---|
| Zoom CSV | one learner on one day (date = file name) | long | `Email` |
| Labs / Quizzes | one learner | wide (`Week 1`..`Week 10`) | `email` |
| Participation | one day; names comma-separated in one cell | packed list | `Name` |
| Status | one learner | long | `email` |

- `Email` is unique per learner (40 Cloud + 100 Power BI = 140); no learner is in both tracks.
- Each Name maps to exactly one Email, so Participation is linked via Name → Email.

## Data quality issues
| # | Issue | Cloud | Power BI | Fix (Power Query) |
|---|---|---|---|---|
| 1 | Zoom Duration format | minutes (`66`) | text `h:mm:ss` (`1:46:00`) | convert both to minutes |
| 2 | Zoom time format | 24-hour (`18:09`) | 12-hour (`06:02 PM`) | convert to Time type |
| 3 | File names differ | `labs_and_quizzes.xlsx` | `Labs & Quizes.xlsx` | load by folder, not file name |
| 4 | Participation sheet name | `Participation` | `Sheet1` | use first sheet by position |
| 5 | Track / date not in columns | – | – | take from folder and file names |

No blanks or duplicate rows were found. Scores of 0 exist (Cloud: 4 labs, 2 quizzes; Power BI: 13 labs, 11 quizzes).

## Decisions
| Brief asks for | Decision |
|---|---|
| Attendance | "Attended" only if Duration **> 30** minutes (exactly 30 = not attended) |
| Cohort | Each track is one cohort |
| Dropout | `Non Graduate` counts as a dropout |
| Certification Type slicer | Left out – not in the data |
| Month/Week filter for assessments | Each weekly score is dated to that week's last class day (from Attendance), e.g. Week 9 → 04-Oct (October) |

## Key facts (for checking the dashboard)
| Metric | Cloud | Power BI | Total |
|---|---|---|---|
| Learners | 40 | 100 | 140 |
| Zoom rows | 2,000 | 5,000 | 7,000 |
| Attendance rate (> 30 min) | 79.25% | 100% | – |
| Graduates | 33 | 86 | 119 |
| Dropouts (Non Graduate) | 7 | 14 | 21 |
| Certified | 24 | 68 | 92 |