---
title: "Common Table Expressions in Active Record"
date: 2023-05-28T09:00:00+10:00
slug: "common-table-expressions-in-active-record"
description: "Taking a look at how to use a Common Table Expression in Active Record queries in Rails 7.1+"
keywords: []
draft: false
tags: [development]
math: false
toc: false
---

Common Table Expressions (CTEs) offer a helpful way to simplify complex database queries but, prior to Rails 7.1, you either needed to write the SQL yourself or drop down to Arel. Arel is private API — so you shouldn't rely upon it — and it can be difficult to read and write because it sits at a lower level of abstraction. Now that they are available in Active Record, let's take a look at CTEs and why you should care.

## What is a Common Table Expression?
A CTE is a named temporary result set that you can reference within a SELECT, INSERT, UPDATE, or DELETE statement. You can think of it as a temporary view that only exists for the duration of your query.

The primary benefit they provide is improving the readability and maintainability of your SQL. Complex queries involving numerous joins, subqueries, or aggregations can become confusing and unwieldy. By breaking down a complex query into simpler parts using CTEs, you can make the query more understandable and easier to debug.

This is particularly useful when working on a team or maintaining code over time. By improving the readability of your queries, you make it easier for other team members to understand your code, making collaboration more efficient. Similarly, when revisiting your own code months or years later, having a clear structure provided by CTEs can help you understand and modify your code more easily. Far too many times I've seen organisations avoid modifying complex queries because they are hard to reason about, and no one wants to be responsible for touching them.

When used in the context of Active Record, they also provide a way to reuse common query parts without managing SQL string manipulation.

## When might you use a CTE?

In most situations where I find myself reaching for a CTE, it's for no other reason than to break a query into logical parts that can be read in isolation. I've always found the context switch when moving to SQL to be quite difficult.

When writing an application, there should be a focus on breaking the business logic into small parts that each represent a single responsibility. When structuring a database query, however, multiple concepts need to be intertwined. To understand the full context of any part of the query, you often need to scan up and down to mentally link an item in the SELECT to its corresponding FROM, JOIN, WHERE, GROUP, etc. I've always found that CTEs allow me to query in a way that is more consistent with my mental model of the application.

Let's say you've been tasked with performing a review of the gender salary gap in your organisation. You will do this by performing a query on the employee table:
```
Name     | Gender | Position        | Salary
--------------------------------------------
Dewayne  | male   | Engineer        | 100000
Jonathon | male   | Engineer        | 105000
Jamey    | male   | Engineer        | 110000
Phil     | male   | Engineer        | 107500
Chester  | male   | Engineer        | 102500
Lavinia  | female | Engineer        |  90000
Patrina  | female | Engineer        |  95000
Reagan   | female | Engineer        | 100000
Candace  | female | Engineer        | 102500
Era      | female | Engineer        |  92500
Morgan   | other  | Engineer        |  91000
Baylor   | other  | Engineer        |  95000
Rolf     | male   | Senior Engineer | 150000
Marvin   | male   | Senior Engineer | 155000
Gregg    | male   | Senior Engineer | 160000
Rufus    | male   | Senior Engineer | 157500
Wesley   | male   | Senior Engineer | 152500
Emilie   | female | Senior Engineer | 140000
Tamar    | female | Senior Engineer | 145000
Iraida   | female | Senior Engineer | 147500
Thuy     | female | Senior Engineer | 150000
Veta     | female | Senior Engineer | 152500
Lennox   | other  | Senior Engineer | 135000
Dakota   | other  | Senior Engineer | 147500
```

You might think about this as two separate tasks.

### Get a list of all employees
```ruby
Employee.all
```

### Average the salaries of employees by position
```ruby
Employee.select(
  :position, "AVG(salary) AS average_salary"
).group(:position)
```

### Putting it together

```ruby
average_salary_by_position = Employee.select(
  :position, "AVG(salary) AS average_salary"
).group(:position)

Employee.with(average_salary_by_position:).
  joins("INNER JOIN average_salary_by_position USING (position)").
  select(
    "employees.*",
    "average_salary_by_position.average_salary"
  ).each do |employee|
    salary_delta = employee.salary - employee.average_salary
    puts "#{employee.name}: #{salary_delta}"
  end

# OUTPUT
# Dewayne: 750.0
# Jonathon: 5750.0
# Jamey: 10750.0
# Phil: 8250.0
# Chester: 3250.0
# Lavinia: -9250.0
# Reagan: 750.0
# Candace: 3250.0
# Era: -6750.0
# Morgan: -8250.0
# Baylor: -4250.0
# Rolf: 625.0
# Marvin: 5625.0
# Gregg: 10625.0
# Rufus: 8125.0
# Wesley: 3125.0
# Emilie: -9375.0
# Tamar: -4375.0
# Iraida: -1875.0
# Thuy: 625.0
# Veta: 3125.0
# Lennox: -14375.0
# Dakota: -1875.0
```

Active Record produces SQL that looks something like:
```SQL
WITH "average_salary_by_position" AS (
  SELECT
    "employees"."position",
    AVG(salary) AS average_salary
  FROM
    "employees"
  GROUP BY
    "employees"."position"
)
SELECT
  employees.*,
  "average_salary_by_position"."average_salary"
FROM
  "employees"
INNER JOIN
  average_salary_by_position USING (position)
```

## Is that actually better?

I admit, I would be unlikely to use a CTE in that case, but I wanted to start with a simple example.

It's quite common for me to produce reports that compare records across tables. In these situations, the separation of logic can make it easier for me to conceptualise.

For example, maybe the business would like a report that identifies customers that have been ordering less this year than in previous years. This information might allow sales representatives to better focus their efforts on at-risk customers.

Without CTEs, you might produce something like:
```ruby
recent_start = Time.current.beginning_of_year
comparison_start = 1.year.ago.beginning_of_year

Customer.joins(
  "INNER JOIN sales_orders AS comparison_orders ON " \
  "customers.id = comparison_orders.customer_id"
).joins(
  "INNER JOIN sales_invoices AS comparison_invoices ON " \
  "comparison_orders.id = comparison_invoices.sales_order_id"
).joins(
  "LEFT JOIN sales_orders AS recent_orders ON " \
  "customers.id = recent_orders.customer_id AND " \
  "recent_orders.date_created >= '#{recent_start.to_fs(:db)}'",
).where(
  comparison_invoices: { created_at: comparison_start..comparison_start.end_of_year }
).group(
  "customers.id"
).select(
  "customers.name",
  "SUM(comparison_invoices.subtotal) AS comparison_sum_subtotal",
  "COALESCE(SUM(recent_orders.subtotal), 0) AS recent_sum_subtotal"
)
```

I find this difficult to follow. There is a heavy cognitive load involved in understanding what the query does and what it will return.

If it is split up using CTEs, however:
```ruby
recent_start = Time.current.beginning_of_year
comparison_start = 1.year.ago.beginning_of_year

recent_orders = SalesOrder.joins(:customer).
  where("sales_orders.created_at > ?", recent_start).
  group(:name).
  select(:name, "SUM(sales_orders.subtotal) AS sum_subtotal")

comparison_invoices = SalesInvoice.joins(order: :customer).
  where(sales_invoices: { created_at: comparison_start..comparison_start.end_of_year }).
  group(:name).
  select(:name, "SUM(sales_invoices.subtotal) AS sum_subtotal")

Customer.with(recent_orders:, comparison_invoices:).
  from("comparison_invoices").
  joins("INNER JOIN recent_orders USING (name)").
  order("comparison_invoices.sum_subtotal DESC").
  select(
    "comparison_invoices.name",
    "comparison_invoices.sum_subtotal AS comparison_sum_subtotal",
    "COALESCE(recent_orders.sum_subtotal, 0) AS recent_sum_subtotal"
  )
```

This combination of labelling the queries and grouping them by task really appeals to me. If you use query objects, each of those components can be placed in a separate public method or query object, allowing them to be reused throughout your application.

## Conclusion
In conclusion, Common Table Expressions are a powerful tool that can simplify complex SQL queries, making them a great asset for any software engineer. They can make your queries more readable, maintainable, and powerful. So, the next time you find yourself facing a complex query or a problem involving hierarchical data, remember the power of CTEs!
