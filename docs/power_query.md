# Power Query (M) – Data Preparation

All queries start from the `DataFolder` parameter → `Files` base query.

## Files (helper, not loaded)
```
let
    // 1. Every file in data\raw and all its subfolders
    Source = Folder.Files(DataFolder),

    // 2. Drop Excel lock files (they start with ~$)
    #"Removed Lock Files" = Table.SelectRows(Source,
        each not Text.StartsWith([Name], "~$")),

    // 3. Folder path without the DataFolder part
    #"Added Relative Path" = Table.AddColumn(#"Removed Lock Files", "Relative Path",
        each Text.Middle([Folder Path], Text.Length(DataFolder) + 1), type text),

    // 4. Track from the first folder name
    #"Added Track" = Table.AddColumn(#"Added Relative Path", "Track",
        each if Text.StartsWith([Relative Path], "Cloud Training") then "AWS Cloud"
        else if Text.StartsWith([Relative Path], "PowerBI Training") then "Power BI"
        else null, type text),

    // 5. Data Type from the second folder name
    #"Added Data Type" = Table.AddColumn(#"Added Track", "Data Type",
        each Text.BeforeDelimiter(Text.AfterDelimiter([Relative Path], "\"), "\"),
        type text),

    // 6. Keep only the columns we need
    #"Selected Columns" = Table.SelectColumns(#"Added Data Type",
        {"Content", "Name", "Folder Path", "Relative Path", "Track", "Data Type"})
in
    #"Selected Columns"
```

## Attendance
```
let
    // 1. Start from the base query
    Source = Files,

    // 2. Keep only the Zoom CSV files
    #"Zoom Files" = Table.SelectRows(Source,
        each [Data Type] = "Zoom Attendance" and Text.EndsWith([Name], ".csv")),

    // 3. Read each CSV into a nested table (all columns as text)
    #"Parsed CSV" = Table.AddColumn(#"Zoom Files", "Data",
        each Table.PromoteHeaders(
            Csv.Document([Content], [Delimiter = ",", Encoding = 65001]),
            [PromoteAllScalars = true])),

    // 4. Date from the file name, e.g. "05-Aug-2024.csv"
    #"Added Date" = Table.AddColumn(#"Parsed CSV", "Date",
        each Date.FromText(Text.BeforeDelimiter([Name], ".csv"),
            [Format = "dd-MMM-yyyy", Culture = "en-US"]), type date),

    // 5. Week number from the folder, e.g. "...\Week 10\"
    #"Added Week" = Table.AddColumn(#"Added Date", "Week",
        each Number.FromText(
            Text.AfterDelimiter(
                Text.BeforeDelimiter(
                    Text.AfterDelimiter([Relative Path], "Zoom Attendance\"), "\"),
                "Week ")), Int64.Type),

    // 6. Keep only what we need before expanding
    #"Kept Columns" = Table.SelectColumns(#"Added Week",
        {"Track", "Date", "Week", "Data"}),

    // 7. Open the nested tables and stack all rows
    #"Expanded Data" = Table.ExpandTableColumn(#"Kept Columns", "Data",
        {"Name", "Email", "Join Time", "Leave Time", "Duration", "Role"}),

    // 8. Duration -> minutes (handles "66" AND "1:46:00")
    #"Added Minutes" = Table.AddColumn(#"Expanded Data", "Minutes",
        each if Text.Contains([Duration], ":")
             then Duration.TotalMinutes(Duration.FromText([Duration]))
             else Number.From([Duration]),
        type number),

    // 9. Remove the old text Duration column
    #"Removed Duration" = Table.RemoveColumns(#"Added Minutes", {"Duration"}),

    // 10. Join/Leave Time text -> Time (works for "18:09" and "06:02 PM")
    #"Typed Times" = Table.TransformColumnTypes(#"Removed Duration",
        {{"Join Time", type time}, {"Leave Time", type time}}, "en-US"),

    // 11. Defensive clean: trim spaces and lower-case emails
    #"Cleaned Email" = Table.TransformColumns(#"Typed Times",
        {{"Email", each Text.Lower(Text.Trim(_)), type text}}),

    // 12. Remaining text columns
    #"Typed Text" = Table.TransformColumnTypes(#"Cleaned Email",
        {{"Name", type text}, {"Role", type text}}),

    // 13. 30-minute rule: Attended only if MORE than 30 minutes
    #"Added Attended" = Table.AddColumn(#"Typed Text", "Attended",
        each if [Minutes] > 30 then 1 else 0, Int64.Type)
in
    #"Added Attended"
```

## Assessments
```
let
    // 1. Start from the base query
    Source = Files,

    // 2. Keep only the Labs & Quizzes Excel files (one per track)
    #"LQ Files" = Table.SelectRows(Source,
        each [Data Type] = "Labs & Quizes" and Text.EndsWith([Name], ".xlsx")),

    // 3. Open each workbook (true = first row is headers)
    #"Opened Workbooks" = Table.AddColumn(#"LQ Files", "Sheets",
        each Excel.Workbook([Content], true)),

    // 4. Keep Track + the list of sheets, then expand the sheets
    #"Kept Columns" = Table.SelectColumns(#"Opened Workbooks", {"Track", "Sheets"}),
    #"Expanded Sheets" = Table.ExpandTableColumn(#"Kept Columns", "Sheets",
        {"Name", "Data", "Kind"}, {"Sheet", "Data", "Kind"}),

    // 5. Only real worksheets (not named ranges)
    #"Sheets Only" = Table.SelectRows(#"Expanded Sheets", each [Kind] = "Sheet"),

    // 6. Friendly type: Labs -> Lab, Quizzes -> Quiz
    #"Added Type" = Table.AddColumn(#"Sheets Only", "Assessment Type",
        each if [Sheet] = "Labs" then "Lab"
        else if [Sheet] = "Quizzes" then "Quiz"
        else null, type text),

    // 7. Unpivot each sheet: Week columns -> rows
    #"Unpivoted" = Table.AddColumn(#"Added Type", "Long",
        each Table.UnpivotOtherColumns([Data], {"email"}, "Week", "Score")),

    // 8. Stack all the unpivoted tables
    #"Kept Long" = Table.SelectColumns(#"Unpivoted", {"Track", "Assessment Type", "Long"}),
    #"Expanded Long" = Table.ExpandTableColumn(#"Kept Long", "Long",
        {"email", "Week", "Score"}, {"Email", "Week", "Score"}),

    // 9. Clean: "Week 3" -> 3, email lower-case, set types
    #"Cleaned" = Table.TransformColumns(#"Expanded Long", {
        {"Week",  each Number.From(Text.AfterDelimiter(_, "Week ")), Int64.Type},
        {"Email", each Text.Lower(Text.Trim(_)), type text},
        {"Score", each Number.From(_), Int64.Type}}),

    // 10. Last class date of each week, per track (from Attendance)
    WeekEnds = Table.Group(Attendance, {"Track", "Week"},
        {{"Date", each List.Max([Date]), type date}}),

    // 11. Give each score the date its week ended
    #"Merged Date" = Table.NestedJoin(#"Cleaned", {"Track", "Week"},
        WeekEnds, {"Track", "Week"}, "W", JoinKind.LeftOuter),
    #"Expanded Date" = Table.ExpandTableColumn(#"Merged Date", "W", {"Date"})
in
    #"Expanded Date"
```

## Participation
```
let
    // 1. Start from the base query
    Source = Files,

    // 2. Keep only the Participation Excel files
    #"Participation Files" = Table.SelectRows(Source,
        each [Data Type] = "Participation" and Text.EndsWith([Name], ".xlsx")),

    // 3. First worksheet of each file (names differ: "Participation" vs "Sheet1")
    #"First Sheet" = Table.AddColumn(#"Participation Files", "Data",
        each Table.SelectRows(Excel.Workbook([Content], true),
            each [Kind] = "Sheet"){0}[Data]),

    // 4. Stack both tracks
    #"Kept Columns" = Table.SelectColumns(#"First Sheet", {"Track", "Data"}),
    #"Expanded Data" = Table.ExpandTableColumn(#"Kept Columns", "Data",
        {"Date", "Participants"}),

    // 5. Split "A, B, C" into a list and trim spaces from each name
    #"Split Names" = Table.AddColumn(#"Expanded Data", "Name",
        each List.Transform(Text.Split([Participants], ","), Text.Trim)),

    // 6. One row per name
    #"Expanded Names" = Table.ExpandListColumn(#"Split Names", "Name"),
    #"Removed Participants" = Table.RemoveColumns(#"Expanded Names", {"Participants"}),

    // 7. Date text -> real date
    #"Typed Date" = Table.TransformColumns(#"Removed Participants",
        {{"Date", each Date.FromText(_, [Format = "dd-MMM-yyyy", Culture = "en-US"]), type date}}),

    // 8. Lookup table: one row per Track + Name + Email (from Attendance)
    Lookup = Table.Distinct(Table.SelectColumns(Attendance, {"Track", "Name", "Email"})),

    // 9. Bring in the email, matching on Track AND Name
    #"Merged Email" = Table.NestedJoin(#"Typed Date", {"Track", "Name"},
        Lookup, {"Track", "Name"}, "Match", JoinKind.LeftOuter),
    #"Expanded Email" = Table.ExpandTableColumn(#"Merged Email", "Match", {"Email"}),

    // 10. Types
    #"Typed Text" = Table.TransformColumnTypes(#"Expanded Email",
        {{"Name", type text}, {"Email", type text}})
in
    #"Typed Text"
```

## Learners
```
let
    // 1. Start from the base query
    Source = Files,

    // 2. Keep only the Status Excel files
    #"Status Files" = Table.SelectRows(Source,
        each [Data Type] = "Status of Learners" and Text.EndsWith([Name], ".xlsx")),

    // 3. First worksheet of each file
    #"First Sheet" = Table.AddColumn(#"Status Files", "Data",
        each Table.SelectRows(Excel.Workbook([Content], true),
            each [Kind] = "Sheet"){0}[Data]),

    // 4. Stack both tracks (rename email -> Email)
    #"Kept Columns" = Table.SelectColumns(#"First Sheet", {"Track", "Data"}),
    #"Expanded Data" = Table.ExpandTableColumn(#"Kept Columns", "Data",
        {"email", "Graduation Status", "Certification Status"},
        {"Email", "Graduation Status", "Certification Status"}),

    // 5. Defensive clean: trim text, lower-case email
    #"Cleaned" = Table.TransformColumns(#"Expanded Data", {
        {"Email", each Text.Lower(Text.Trim(_)), type text},
        {"Graduation Status", Text.Trim, type text},
        {"Certification Status", Text.Trim, type text}}),

    // 6. Bring in the learner's Name from Attendance
    Names = Table.Distinct(Table.SelectColumns(Attendance, {"Track", "Name", "Email"})),
    #"Merged Name" = Table.NestedJoin(#"Cleaned", {"Track", "Email"},
        Names, {"Track", "Email"}, "N", JoinKind.LeftOuter),
    #"Expanded Name" = Table.ExpandTableColumn(#"Merged Name", "N", {"Name"}),

    // 7. Dropout rule: Non Graduate = dropout
    #"Added Dropout" = Table.AddColumn(#"Expanded Name", "Dropout",
        each if [Graduation Status] = "Non Graduate" then "Yes" else "No", type text),

    // 8. Program end date per track = last attendance date
    EndDates = Table.Group(Attendance, {"Track"},
        {{"Program End", each List.Max([Date]), type date}}),
    #"Merged End" = Table.NestedJoin(#"Added Dropout", {"Track"},
        EndDates, {"Track"}, "E", JoinKind.LeftOuter),
    #"Expanded End" = Table.ExpandTableColumn(#"Merged End", "E", {"Program End"}),

    // 9. Program Status: finished before today = Completed
    #"Added Program Status" = Table.AddColumn(#"Expanded End", "Program Status",
        each if [Program End] < Date.From(DateTime.LocalNow())
             then "Completed" else "Ongoing", type text),

    // 10. Final column order and types
    #"Selected Columns" = Table.SelectColumns(#"Added Program Status",
        {"Email", "Name", "Track", "Graduation Status", "Certification Status",
         "Dropout", "Program End", "Program Status"}),
    #"Typed" = Table.TransformColumnTypes(#"Selected Columns",
        {{"Name", type text}, {"Program End", type date}})
in
    #"Typed"
```