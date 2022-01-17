---
layout: post
title: Solving Sudoku with a Single SQL Query, Part 2
---

In this post, I'll be diving deep into the code that was presented in [part 1]({% post_url 2016-07-06-sudoku %}).

## Recursive CTEs

My solution utilizes a lot of recursive common table expressions (CTEs). CTEs are a simple way of defining a temporary projection in T-SQL that can be referenced multiple times later on in a query. Their real power, however, lies in their ability to be self-referencing and thus recursive. In a recursive CTE, we define an initial projection and UNION ALL that with a projection that references the table currently being defined.

Since this puzzle is all about digits 1 through 9, I need to make something that will get me those values to work with.

```sql
;WITH Cnt AS 
(
   SELECT 1 AS [val]
     UNION ALL
   SELECT [val] + 1
   FROM Cnt -- This is the table I'm creating right now, A SELF-REFERENCING DEFINITION!!!
   WHERE [val] < 9
)
SELECT * FROM Cnt
```

This is an easy way to get the digits we need without having to list them all out. The first statement in the expression gives the basis of the recursion (1), then we `UNION ALL` to the Cnt table, selecting the value in the table + 1. Initially Cnt has a single record with [val] = 1. Then we union that with a record that is [val] + 1 = 2. This continues until the constraint on the second selection is no longer true. Once [val] = 9, we cannot union it again and we end up with a table of values 1 through 9.

## Assigning Each Cell

Next, I want to get a set of all the possible locations in the grid with values ranging from 1 to 81. I could do another CTE like the one I just did, but I already have values 1 to 9, I can utilize my Cnt table to create my location table.

```sql
locations
  AS (
          SELECT    [row].[val] AS [row]
                   ,[col].[val] AS [col]
                   ,(([row].[val] - 1) * 9) + [col].[val] AS [loc]
          FROM      Cnt AS [row]
          CROSS APPLY ( val FROM [Cnt] ) AS [col]
         ),
```

Locations Table Contents

| row | col | loc |
| --- | --- | --- |
| 1   | 1   | 1   |
| 1   | 2   | 2   |
|     | ... |     |
| 9   | 8   | 80  |
| 9   | 9   | 81  |

## Assigning Cells to Groups

In the game of Sudoku, each cell contains a digit that is unique within the row, column, and square that the cell is a member of. In order to facilitate checking that the value is unique, we will create a table that defines all the groups that each cell is a member of. The row and column groups are fairly easy to define, but the nine 3x3 boxes are a bit more difficult. In order to better set these groupings apart, I prefix the <em>row groupings with a 10</em>, the <em>column groupings with a 20</em> and the <em>box groupings with a 30</em>.

This will allow us to cross-reference any locations to see if a particular value already exists. For instance, if I'm looking at the value in location 19, I'll need to look at all the locations that are part of the groupings 103, 201, and 301 to see if the value I want to put in that location is already in any of the other locations in those groups.

```sql
groups
  AS (
      SELECT   -- Groups of Rows
                FLOOR(([loc] - 1) / 9) + 101 AS [grouping]
               ,[loc]
      FROM      [locations]
      UNION
      SELECT   -- Groups of Columns
                (([loc] - 1) % 9) + 201 AS [grouping]
               ,loc
      FROM      [locations]
      UNION
      SELECT   -- Groups of 3x3 Squares
                FLOOR(([row] - 1) / 3) * 3 + FLOOR(([col] - 1) / 3) + 301 [grouping]
               ,[locations].[loc]
      FROM      locations
     ),
```

The first two select clauses are retrieving the row and column groupings respectively, then the third grouping is creating groups for the nine 3x3 boxes in the grid. With these available to us, the job of building up a valid solution will be much easier.

| grouping | loc           |
| -------- | ------------- |
| 101      | 1  -- Rows    |
| 101      | 2             |
| ...      |               |
| 101      | 9             |
| 102      | 10            |
| ..       |               |
| 102      | 18            |
| 103      | 19            |
| ...      |               |
| 109      | 81            |
| 201      | 1  -- Columns |
| 201      | 10            |
| 201      | 19            |
| 201      | 28            |
| 201      | 37            |
| 201      | 46            |
| 201      | 55            |
| 201      | 64            |
| 201      | 73            |
| 202      | 2             |
| ...      |               |
| 301      | 1  -- Squares |
| 301      | 2             |
| 301      | 3             |
| 301      | 10            |
| 301      | 11            |
| 301      | 12            |
| 301      | 19            |
| 301      | 20            |
| 301      | 21            |
| 302      | 4             |
| ...      |               |
