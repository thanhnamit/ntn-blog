---
featured: false
title: "Postgresql text search functions"
tags: [postgresql, apis]
date: 2019-05-10T14:24:10+10:00
lastmod: 2019-05-10T14:24:10+10:00
---

![Postgresql cool](/img/posts/2019-05-10-postgresql.png 'Postgresql cool')

In one of my current projects, I built a search API backed by a relational database with the dataset size of 2.5 million address records.
the API is supposed to return a list of matching addresses given address details such as house number, street name, postcode, etc. If there is a single
address matching the input, the API should only respond with one record. The data model is simple:

- addressLine
- flatOrUnitNumber
- unitSuffix
- floorOrLevelNumber
- buildingNumber
- buildingNumberFirst
- buildingNumberLast
- streetName
- streetType 
- streetTypeSuffix
- suburbOrPlaceOrLocality
- stateOrTerritory
- postCode


The problem seems to be trivial given the structure above; the table contains a full address line and specific details in each column. It looks like
a simple SQL query with comparisons such as `LIKE`, `=` should just work.

Unfortunately, when examining the data rows, the data quality is not so great, e.g. streetType can be `Rd`, `rd`, and `Road`. This makes the problem
a little bit harder to solve with the simple query approach, and the outcome is not reliable unless extra data cleansing the data effort is in place.

### Enter Postgresql 

There are many ways to skin a cat, e.g. indexing data with Elastic Search engine and utilising its powerful full-text search capability, but this requires extra
effort. Luckily, during my study, I have stumbled upon really cool Postgresql's function duet: `ts_tovector` and `ts_query` ([Text Search Functions and Operators](https://www.postgresql.org/docs/9.6/functions-textsearch.html)) which nicely solved my problem for the reasons that the quality of `addressLine` field is consistent and no need to update or insert records on a regular basis.

To use these functions:

```sql
select * from db where to_tsvector(addressLine) @@ to_tsquery('5 & 14 & bourke & (oakleigh | clayton) & 3166')
```
<pre></pre>
Function `to_tsvector` tokenises and normalises the field into words, and each word is a lexeme (unit of lexical meaning or root of word, e.g. required -> require). The normalisation
is configurable in Postgresql.

Function `to_tsquery` performs the search; with parameter in form of a query including keywords and operators such as `OR (|), AND (&), NEGATE (!)`. Keywords can be grouped in whatever way
you like.

At first glance, I was skeptical about performance of this query on my dataset, but it turned out to be quite good, around 100-300ms per 2.5 million records. It was not fast enough when
combined with other part of the query, but a good trick is to pre-index it and store in another column. So I achieved better performance with the improved query below.

```sql
select * from db where (address_vectorised @@ to_tsquery('95 & 305'))
```
<pre></pre>
Happy searching! :feet:

