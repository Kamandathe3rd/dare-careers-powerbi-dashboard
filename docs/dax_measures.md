# DAX Measures

All measures live in the `_Measures` table, grouped in display folders.
Base measures are reused (e.g. `[Total Learners]`) so each rule is written once.

## Code
```dax
DEFINE
    MEASURE '_Measures'[Total Learners] = COUNTROWS ( Learners )
    MEASURE '_Measures'[Total Graduations] = CALCULATE (
    [Total Learners],
    KEEPFILTERS ( Learners[Graduation Status] = "Graduate" )
)
    MEASURE '_Measures'[Total Certifications] = CALCULATE (
    [Total Learners],
    KEEPFILTERS ( Learners[Certification Status] = "Certified" )
)
    MEASURE '_Measures'[Total Dropouts] = CALCULATE (
    [Total Learners],
    KEEPFILTERS ( Learners[Dropout] = "Yes" )
)
    MEASURE '_Measures'[Graduation Rate] = DIVIDE ( [Total Graduations], [Total Learners] )
    MEASURE '_Measures'[Certification Rate] = DIVIDE ( [Total Certifications], [Total Learners] )
    MEASURE '_Measures'[Dropout Rate] = DIVIDE ( [Total Dropouts], [Total Learners] )
    MEASURE '_Measures'[Total Sessions] = COUNTROWS ( Attendance )
    MEASURE '_Measures'[Sessions Attended] = SUM ( Attendance[Attended] )
    MEASURE '_Measures'[Attendance Rate] = DIVIDE ( [Sessions Attended], [Total Sessions] )
    MEASURE '_Measures'[Total Hours in Class] = VAR AttendedMinutes =
    CALCULATE (
        SUM ( Attendance[Minutes] ),
        KEEPFILTERS ( Attendance[Attended] = 1 )
    )
RETURN
    DIVIDE ( AttendedMinutes, 60 )
    MEASURE '_Measures'[Avg Hours per Learner] = DIVIDE ( [Total Hours in Class], [Total Learners] )
    MEASURE '_Measures'[Participation Days] = COUNTROWS ( Participation )
    MEASURE '_Measures'[Participation Rate] = DIVIDE ( [Participation Days], [Total Sessions] )
    MEASURE '_Measures'[Avg Assessment Score] = AVERAGE ( Assessments[Score] )
    MEASURE '_Measures'[Avg Quiz Score] = CALCULATE (
    [Avg Assessment Score],
    KEEPFILTERS ( Assessments[Assessment Type] = "Quiz" )
)
    MEASURE '_Measures'[Avg Lab Score] = CALCULATE (
    [Avg Assessment Score],
    KEEPFILTERS ( Assessments[Assessment Type] = "Lab" )
)
    MEASURE '_Measures'[Labs Completed] = CALCULATE (
    COUNTROWS ( Assessments ),
    KEEPFILTERS ( Assessments[Assessment Type] = "Lab" ),
    KEEPFILTERS ( Assessments[Score] > 0 )
)
    MEASURE '_Measures'[Avg Labs per Learner] = DIVIDE ( [Labs Completed], [Total Learners] )

EVALUATE
    SUMMARIZECOLUMNS(
        "Total Learners", [Total Learners],
        "Total Graduations", [Total Graduations],
        "Total Certifications", [Total Certifications],
        "Total Dropouts", [Total Dropouts],
        "Graduation Rate", [Graduation Rate],
        "Certification Rate", [Certification Rate],
        "Dropout Rate", [Dropout Rate],
        "Total Sessions", [Total Sessions],
        "Sessions Attended", [Sessions Attended],
        "Attendance Rate", [Attendance Rate],
        "Total Hours in Class", [Total Hours in Class],
        "Avg Hours per Learner", [Avg Hours per Learner],
        "Participation Days", [Participation Days],
        "Participation Rate", [Participation Rate],
        "Avg Assessment Score", [Avg Assessment Score],
        "Avg Quiz Score", [Avg Quiz Score],
        "Avg Lab Score", [Avg Lab Score],
        "Labs Completed", [Labs Completed],
        "Avg Labs per Learner", [Avg Labs per Learner]
    )
```

## Expected results (verified)
| Measure | AWS Cloud | Power BI | Total |
|---|---|---|---|
| Total Learners | 40 | 100 | 140 |
| Total Graduations | 33 | 86 | 119 |
| Total Certifications | 24 | 68 | 92 |
| Total Dropouts | 2 | 0 | 2 |
| Graduation Rate | 82.50% | 86.00% | 85.00% |
| Certification Rate | 60.00% | 68.00% | 65.71% |
| Dropout Rate | 5.00% | 0.00% | 1.43% |
| Total Sessions | 2,000 | 5,000 | 7,000 |
| Sessions Attended | 1,585 | 5,000 | 6,585 |
| Attendance Rate | 79.25% | 100.00% | 94.07% |
| Total Hours in Class | 1,813.02 | 8,752.68 | 10,565.70 |
| Avg Hours per Learner | 45.33 | 87.53 | 75.47 |
| Participation Days | 1,784 | 4,253 | 6,037 |
| Participation Rate | 89.20% | 85.06% | 86.24% |
| Avg Assessment Score | 51.85 | 50.40 | 50.81 |
| Avg Quiz Score | 52.75 | 51.04 | 51.53 |
| Avg Lab Score | 50.94 | 49.75 | 50.09 |
| Labs Completed | 396 | 987 | 1,383 |
| Avg Labs per Learner | 9.90 | 9.87 | 9.88 |