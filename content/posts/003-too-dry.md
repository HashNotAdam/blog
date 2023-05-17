---
title: "Too DRY (I should have repeated myself)"
date: 2020-07-25T17:54:58+10:00
slug: "too-dry"
aliases:
  - /blog/too-dry
description: ""
keywords: []
draft: false
tags: [development]
math: false
toc: false
---
"Don't repeat yourself" is a fundamental principle of modern application
development, and with good reason. Duplicating logic increases both the cost of
maintainability and the likelihood of error but I would argue that we are often
too dogmatic in our approach.

There is a common theory that you should wait until your second duplication (or
third instance) to create an abstraction. It would cost more to deal with an
inappropriate abstraction than it would to keep two pieces of code in sync. Even
when following this policy, however, it’s easy to be too DRY.

Tests are a great example of code that can often be made better from being WET.
It’s difficult enough to debug a quirky test without also trying to inspect
assertions in a loop or scrolling around the page to understand what parameters
are being defined.

This week, despite knowing better, I fell for the DRY trap and it was painful.

We have an internal application that uses an API to pull sales and purchase
records from our Enterprise Resource Planning (ERP) system for the purpose of
producing custom reports. The structure of this data is just as boring as one
might imagine: there are sales orders, invoices, and credit notes; and purchase
orders, invoices, and credit notes.

Previously, we have not needed to store the items on an order. Not only were we
only reporting on invoiced orders but our ERP has this amazing feature whereby
there is absolutely no connection between the line items on an order and those
on an invoice other than the fact that they will usually show the same products.
🤷‍♂‍

We would like the system to start predicting when we are going to reach our
credit limits with suppliers and warn us so we can pay down the account before
supply is cut. This means we now need to know about orders otherwise our
reporting will always be retrospective. We are bringing in both sales and
purchase orders but, for simplicity, I’m just going to focus on sales. Here is a
simplified version of the table structure:

![](/blog/003-too-dry/database-before-refactor.svg)

The duplication here lies in all the “lines” tables. They are essentially
identical other than the name of the foreign key (`sales_order_id`,
`sales_invoice_id`, `sales_credit_note_id`). This looks like something that
could be DRYed up. Even if you follow the rule that you create an abstraction
after the second duplication, this would seem like a candidate for refactoring.

To condense this down from 3 to 1 tables, however, we need some way of dealing
with those varying named foreign keys. When a column of a table has
relationships with multiple other tables, it is said to have a polymorphic
association.
[Ruby on Rails actually handles those really well](https://guides.rubyonrails.org/association_basics.html#polymorphic-associations).
In short, the table ends up having 2 columns for the association. The first
column, `*_id`, is the foreign key that existed in the previous tables while the
second, `*_type`, stores the class name of the associated record (e.g.
SalesOrder, SalesInvoice, or SalesCreditNote).

Our new table stores all the line items for sales tables, so we could call it
`sales_lines` but we also need to give a name to the relationship. When defining
one-to-many relationships in Rails/Active Record, we would normally use the
`belongs_to` method on the class that is storing the foreign key. In this case,
that would be the `sales_lines` table but not only would that require defining
all the possible relationships on the polymorphic class but, when using
`belongs_to`, Active Record will assume there is a column in the database named
like `foreign_table_id`.

To deal with this issue, we need to give the polymorphic relationship a name and
then reference that in the foreign classes. Naming is hard… frustratingly so. We
are storing a list of line items and the Rails convention is to apply the suffix
"able" to duck types/polymorphs so we can cringe and call it “itemable”. That
means our table needs to have `itemable_id` and `itemable_type` columns. Note:
if your ID is a standard sequential integer, your Active Record migration can
simply say:
```ruby
t.references :itemable, polymorphic: true, null: false
```

Active Record knows to create both columns and also creates a multi-column
index. After the refactor, the database looks like this:

![](/blog/003-too-dry/database-after-refactor.svg)

At this point, you might pat yourself on the back and bask in all the
duplication you've removed. Before you can commit and show off your amazing work
to your team, you need to run your test suite. As a sea of red floods your
screen, your reality dawns on you and your heart sinks.

The first problem you’ve got is that your application has references to
SalesInvoiceLine and SalesCreditNoteLine. Easy, right‽ Just do a
find-and-replace. Except the association names will also be incorrect. So, when
saving a line for a SalesInvoice, you may have had:
```ruby
SalesInvoiceLine.create!(
  …
  sales_invoice: invoice
)
```
But now that needs to be:
```ruby
SalesLine.create!(
  …
  itemable: invoice
)
```
Then there is the issue of your factories or fixtures. Even if you create a
generic SalesLine factory, you either need to assign it traits based on the
associated records or you need to define that association in your tests. Either
way, you’re going to need to touch every test that refers to them.

Once you’ve wasted hours cleaning all that up and you’re all green again, you
need to ask yourself “what have I really achieved here?”

If we go back to the point of DRY, it reduces the cost of maintainability.
Merging those tables would be valuable as long as a change to any one of them
would necessarily need to be made on the other two but that is far from certain.
While our ERP uses the same class for all lines—and we can therefore anticipate
that a change in their system would require a change to all three tables—if we
decide we need a new column for our own use only for one type of line, we are
forced to apply that column to all types.

So how are you supposed to know when to DRY and when duplication is acceptable?
Outside of tests, you need to ask “are these things the same or are they merely
similar?”

While a line on an order and a line on an invoice _look_ the same, they
are not actually the same. A change in the structure of an order line does not
necessarily require that a change be make to an invoice line. On the other hand,
if you have a method calculating tax in the SalesOrderLine, SalesInvoiceLine,
and SalesCreditNoteLine classes, that is duplication because the concept of tax
and how it is calculated does not change depending upon the sales lifecycle.

While the urge to de-duplicate can be great, applying the same-or-similar test
can save you creating an unnecessary abstraction that increases, rather than
decreases, maintainability.
