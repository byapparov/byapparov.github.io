---
layout: post
title: "Declarative Data Quality with R"
date: 2018-06-09 16:16:01 -0600
categories: [post, package]
tags: [package, rdqa, r]
---

Data Quality initiatives are usually associated with a lot of mystery. In IT systems there are always known and new data problems that constantly hinder data analysis and block insight delivery.

Quite often these problems can and should be solved in the source systems
through stricter data schema. In cases where it is not possible people have to review data problems regularly and update corresponding records.

Approach that I find useful is communicating data problems to stakeholders on a regular basis. [rdqa](https://github.com/byapparov/rdqa) package provides means to data validation through declarative rules with the output that can be shared for correction or further investigation.

In this post I will cover the basic data rules that can be implemented with `rdqa` to allow for a concise business rules definition.

To understand the benefits of `rdqa` approach lets see how identification of duplicates can be implemented with declarative vs imperative code.

{% highlight SQL %}
-- Declarative constraint in the relational database
CREATE TABLE customers (
    email VARCHAR(255),
    UNIQUE(email)
)
{% endhighlight %}
This looks nice and simple and if I use a list in R to represent it turns into something like this:
{% highlight R %}
# Declarative constraint analogue in R object
schema <- list(
  table = "customers",
  column = "email",
  unique = TRUE
)
class(schema) <- "Schema"
{% endhighlight %}

Imperative implementation will require me to use some functions over the columns in my dataset to filter out the errors:
{% highlight R %}
# Imperative processing of duplicates
emails <- read.csv("customers.csv") # we need to load the data
duplicate.emails <- customers[duplicated(emails$email)]
write.csv(duplicate.emails, "duplicate.cusomers.csv")
{% endhighlight %}

Comparing the code above it easy to imagine how difficult the imperative processing of data rules will be as the number of rules grows.

Solution to this is an engine that will automatically process collection of
data rule declarations over a given dataset.

Here are the types of basic column rules available in `rdqa`:

| rule name | description | declaration example |
|-------|--------|---------|
| class | checks column against base R class | class = "integer" |
| unique | finds duplicates in the given column | unique = TRUE |
| required | finds records with missing values in column | required = TRUE |
| regex | validates character column agains a given pattern | regex = "\\w" |
| enum | column matches vector of given values | enum = c("male", "female") |

Rules for the dataset are expressed as a single schema definition:

{% highlight R %}
# file: customers.schema.R
# devtools::install_github('byapparov/rdqa')
library(rdqa)
library(data.table) # rdqa uses data.table package
customers.schema <- Schema(
"customer.data",
  schema = list(
    list(
      name = "id",
      description = "This is an integer primary key for our customer table",
      class = "integer",
      required = TRUE,
      unique = TRUE
    ),
    list(
      name = "name",
      class = "character",
      regex = "\\w"
    ),
    list(
      name = "gender",
      class = "character",
      enum = c("male", "female")
    )
  )
)
{% endhighlight %}
{% highlight R %}
# file: customers.validate.R
source("R/customers.schema.R")
errors <- validate(customers.schema, customers)
print(errors)
{% endhighlight %}

If your project processes data from several datasets it is easy to
organise your code in a way that data validation logic is separated from the main code:

{% highlight bash %}
# code separation simplifies maintenance
R/
- analysis.R
- customers.validate.R
- customers.schema.R
- products.validate.R
- products.schema.R
...
{% endhighlight %}

In the next post I will cover reporting on errors in data.
