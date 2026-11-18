# Progress

## Totals
```dataview
TABLE WITHOUT ID length(rows) AS Boxes, sum(rows.hours) AS Hours, sum(rows.hints) AS Hints
FROM "02-Boxes"
WHERE type = "box"
GROUP BY true
```

## Boxes
```dataview
TABLE date, os, difficulty, status, hours, hints, confidence, published
FROM "02-Boxes"
WHERE type = "box"
SORT date DESC
```

## Redo queue (confidence 1-2)
```dataview
TABLE date, confidence
FROM "02-Boxes"
WHERE type = "box" AND confidence <= 2
SORT confidence ASC
```

## Ready to blog (retired, not yet published)
```dataview
LIST
FROM "02-Boxes"
WHERE type = "box" AND retired = true AND published = false AND status = "done"
```

## Academy modules
```dataview
TABLE date, status, hours, confidence, no-notes-test
FROM "01-Academy"
WHERE type = "module"
SORT date DESC
```
