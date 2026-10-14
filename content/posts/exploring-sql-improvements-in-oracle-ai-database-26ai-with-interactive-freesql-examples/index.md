---
title: "Exploring SQL Improvements in Oracle AI Database 26ai with Interactive FreeSQL Examples"
date: 2026-03-09T07:00:00-03:00
draft: false
categories: ["Oracle 26Ai", "Oracle DBA", "SQL & PL/SQL"]
tags: ["26ai", "Oracle", "Script Oracle"]
cover:
  image: "freesql.png"
---

Oracle AI Database 26ai introduces several SQL improvements designed to simplify common query patterns and make analytical queries easier to write.

Instead of showing static code snippets, this article demonstrates these features using [FreeSQL](https://freesql.com), an interactive SQL environment running Oracle AI Database 26ai.

Each example below can be executed directly in the browser. This makes it possible to experiment with the queries, modify them, and immediately see the results.

The examples use the **OLYM** sample schema available on FreeSQL, which contains Olympic historical data such as athletes, events, nations and medal results.

## Confirming the Database Version

Before exploring the features, we can confirm the environment is running Oracle AI Database 26ai:

The banner returned by the query confirms we are running Oracle AI Database 26ai.

## Running SQL Interactively with FreeSQL

FreeSQL provides a hosted SQL environment running Oracle AI Database 26ai directly in the browser. Each query in this article is embedded using the FreeSQL interactive editor.

This allows readers to run the queries, inspect results, and experiment with the SQL without installing any database locally.

The embedded worksheets behave like a lightweight SQL client: you can execute queries, view result sets, and modify the statements to explore the dataset further.

## Feature 1: SELECT Without FROM

Oracle traditionally required `FROM DUAL` when evaluating expressions. Oracle 26ai allows SELECT expressions without referencing a table.

**Why this matters:** it removes one of the small but persistent Oracle differences that often appeared in examples and cross-platform SQL documentation.

This change also improves compatibility with SQL examples written for other database systems.

## Feature 2: UUID() Function

Oracle 26ai introduces the `UUID()` function which generates RFC 9562 compliant version 4 UUIDs.

**Why this matters:** it provides a native way to generate globally unique identifiers directly from SQL.

This is particularly useful in distributed systems and microservice architectures where unique identifiers must be generated independently.

## Feature 3: Medal Distribution

Using the Olympic dataset we can quickly analyze medal distribution:

**Why this matters:** aggregation queries are a fundamental pattern in analytical SQL. Establishing this baseline makes it easier to understand the ranking and filtering techniques introduced in the next examples.

## Feature 4: Events With the Highest Medal Counts

By joining the medals and events tables we can identify which events produced the highest number of medals.

**Why this matters:** joins and aggregations are fundamental building blocks for analytical queries. Patterns like joining tables, grouping data, and ordering by aggregated results appear frequently in reporting and data exploration workloads.

## Feature 5: QUALIFY Clause

The `QUALIFY` clause is one of the most interesting SQL additions in Oracle 26ai. It allows filtering results of window functions without requiring a subquery.

Below we rank events by medal count and return the top results:

**Why this matters:** this feature removes the need for nested queries when filtering window function results, resulting in simpler and more readable SQL.

## Feature 6: Experimenting with the Dataset

Because the queries are embedded using FreeSQL, readers can easily modify the statements and explore the dataset further.

For example, you can adjust filters, change aggregations, or explore different tables in the OLYM schema to understand how the Olympic dataset is structured.

This makes the article not only a demonstration of SQL features, but also a small interactive playground for experimenting with Oracle AI Database 26ai.

## Quick Reference Summary

| Feature | Description | Introduced |
| --- | --- | --- |
| SELECT without FROM | Allows expressions to be evaluated without referencing DUAL. | 26ai |
| UUID() | Generates RFC 9562 compliant version-4 UUID values directly from SQL. | 26ai |
| QUALIFY | Filters window function results without requiring nested subqueries. | 26ai |
| Interactive SQL | Queries embedded using FreeSQL allow readers to execute SQL directly in the browser. | FreeSQL |

## Summary

These examples use the OLYM sample schema available on FreeSQL.

Oracle AI Database 26ai brings practical SQL improvements, not just cosmetic changes. `SELECT` without `FROM DUAL` removes a long-standing Oracle-specific quirk, `UUID()` adds a native UUID generator, and `QUALIFY` simplifies filtering of analytic functions.

Using FreeSQL makes it possible to run each example directly in the browser, which helps illustrate how these features behave in practice.

*All examples in this article were executed on Oracle AI Database 26ai using the FreeSQL embedded environment.*
