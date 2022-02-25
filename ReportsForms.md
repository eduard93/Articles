# Forms in InterSystems Reports

Recently I needed to to create a report in InterSystems Reports, which was not the usual _table_, but rather a form, to be specific a bill (but it does not matter that it was a bill specifically, just that it was not a _table_). This article deals with the case where you need to create a report which is an invoice a bill or something of a similar nature. 

I would also describe several tricks, which might come useful.

# Data model

The main difference with tha data model (comperad to the normal table or chart reports) is that you need to use queries of a different cardinality together.

In our example we will use two tables - customers and items. We will create a billing report, where each customer has a separate bill and each bill has a varying number of items.

Here's our data model:



N queries - 1 base query and items queries.

# Connecting item queries

## Several data panes and overlap

# Formulas

## Builtins and Java

## Example: visibility

## Example: length

# Vertical text

# Summary

# Links
