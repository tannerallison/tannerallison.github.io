---
layout: post
title: Solving Sudoku with a Single SQL Query, Part 3
categories: SQL, Programming
---

> See the other parts of this discussion [here]({% post_url 2016-07-06-sudoku %}) and [here]({% post_url 2016-07-12-sudoku-part2 %}).

Okay, now that we have the general framework we can start writing the pieces that deal with the values that were given to us in the puzzle. The first thing I'm going to do is create a table that contains each possible value for each cell based on the values that are already in the solution.

```sql
Possibilities
  AS (
      SELECT    [g].[Row] -- first select just sets the value for the given locations.
               ,[g].[Col]
               ,[locations].[loc]
               ,[g].[Data]
      FROM      @given AS [g]
      JOIN      [locations]
                ON [locations].[col] = [g].[Col]
                   AND [locations].[row] = [g].[Row]
      UNION ALL
      SELECT    [possibility].[row]
               ,[possibility].[col]
               ,[possibility].[loc]
               ,[possibility].[val]
      FROM      (
                 SELECT [locations].*
                       ,[possibleValue].[val]
                 FROM   [locations] -- 
                 LEFT OUTER JOIN @given AS [g] -- I join to the given values to exclude these below
                        ON [g].[Col] = [locations].[col]
                           AND [g].[Row] = [locations].[row]
                 CROSS APPLY (
                              SELECT * FROM [Cnt] -- Join to all values 1-9
                             ) AS [possibleValue]
                 WHERE  [g].[Data] IS NULL --Exclude cells already defined in the given set
                ) AS [possibility]
      WHERE     [possibility].[val] NOT IN -- At this point, every cell not in the given set is
            (
                SELECT  [existingCell].[Data]
                FROM    (
                         SELECT [locations].[loc]
                               ,[g].[Data]
                         FROM   @given AS [g]
                         JOIN   [locations]
                                ON [locations].[row] = [g].[Row]
                                   AND [locations].[col] = [g].[Col]
                        ) AS [existingCell]
                JOIN    [groups] [existingCellGroup]
                        ON [existingCellGroup].[loc] = [existingCell].[loc]
                JOIN    [groups] [possibleValueGroup]
                        ON [existingCellGroup].[grouping] = [possibleValueGroup].[grouping]
                           AND [existingCellGroup].[loc] != [possibleValueGroup].[loc]
                WHERE   [possibleValueGroup].[loc] = [possibility].[loc]
            )
     ),
```

First we get all the cells that are already given to us. These cells will only have one valid option. Next we need to union those with all the possible values for all the other cells. We could just include all values 1 thru 9 for each of these cells, but we already know that some values will be impossible because they already exist in a group that the cell belongs to. So to reduce our sample space, we do a check to see if the value already exists in any of the cells that are a member of a group that our cell is a member of.

```sql
rowStrings
  AS (
      SELECT    CAST([Possibilities].[Data] AS VARCHAR(MAX)) AS string
               ,[Possibilities].[loc]
      FROM      [Possibilities]
      WHERE     [Possibilities].[loc] = 1
      UNION ALL
      SELECT    [rowStrings].[string] + '' + CONVERT(VARCHAR(1), [Possibilities].[Data])
               ,[Possibilities].[loc]
      FROM      (
                 SELECT * FROM [rowStrings] WHERE LEN ([string]) = [loc]
                ) AS [rowStrings]
      JOIN      [Possibilities]
                ON [rowStrings].[loc] + 1 = [Possibilities].[loc]
      WHERE     [Possibilities].[Data] NOT IN 
            (
                SELECT   SUBSTRING([rowStrings].[string], [g1].[loc], 1)
                FROM     [groups] [g1]
                JOIN     [groups] [g2]
                         ON [g2].[grouping] = [g1].[grouping]
                            AND [g2].[loc] = [Possibilities].[loc]
                            AND [g1].[loc] &lt; [Possibilities].[loc]
            )
     ),
```
