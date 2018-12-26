---
title: "Converting Looker PDTs to dbt models: Reasons why (and why not)"
date: "2018-12-25"
author: "Dylan Baker"
---

Over the past few years, Looker has become a market-leading BI tool. Its data governance and self-service capabilities have now become paramount to thousands of BI stacks. For many, the Third Wave of BI (TM Frank Bien) is well and truly here.

Persistent derived tables (PDTs) are a big part of Looker's success. They allow businesses to quickly transform their data with simple SQL `select` statement, abstracting lots of the data engineering that might otherwise be required. They allow you to easily materialize those models as tables in your data warehouse, speeding up the query times for your end users. 

They are addictive. Build a few PDTs, define an explore. Push. Build a few more. Push. Re-define the calculation behind a dimension. Push. Like most addictions though, there comes a point where you realise it may be detrimental to your health and well-being. Well, that of your BI stack anyway. 

Models can start getting slow, without the ability to optimise fully. Models can get repetitive, with very portions of code written out over and over. Models can break, without the ability to test their output.

## Enter: dbt.

dbt is to PDTs what Reeses is to peanut butter slathered on store-brand chocolate: broadly the same thing, but one far better executed that the other.

From Tristan Handy, the CEO and founder of Fishtown Analytics who build dbt:

> dbt is the T in ELT. It doesn’t extract or load data, but it’s extremely good at transforming data that’s already loaded into your warehouse. This “transform after load” architecture is becoming known as ELT (extract, load, transform).
> Every model is exactly one SELECT query, and this query defines the resulting data set.

Sounds pretty similar, right? It is. dbt simply allows you to abide by certain principals that PDTs don't. dbt allows you some flexibity that PDTs don't.

In this article, I'm going to cover some of the issues with doing all of the transformative modelling in Looker. I'll describe the pitfalls that may befall you and I'll lay out an argument for why dbt may be a better option for some of your modelling. In the next part of this series, I'll outline the practical steps of actually moving your PDTs into dbt.

## A quick note

Before I go any further, I want to remark that I wholeheartedly believe that Looker is a fantastic BI tool. It has been transformative for my work at a number of different businesses. It has powered most of the reporting I have done in the past three years. It is great. I fully intend to work with Looker for years to come. I wholly endorse it. 

It excels at data governance. It excels at self-service. It excels at integrating with other tools. It's just not the best tool for certain types of data modelling.

It may also be right for your business's modelling _right now_. It's a great way to get started with data modelling if you've never done it before. It removes a few steps of complication that having another tool in the mix adds.

So, with that said, here are the instances where I would recommend a move to dbt and the instances where Looker is going to better serve your needs.

## Performance problems and incremental models

Incredibly often, the issue that brings people from PDTs to dbt is the performance of their models. Most of the time, this stems from one of two problems.

The **first** is in inability to [build PDTs incrementally](https://discourse.looker.com/t/incremental-pdts/7620). 

Imagine you've recently set up Heap, Segment, Snowplow or some other solution for event tracking. Initially, you've only got a few days of data, then a few weeks, then a few months. Every time your PDT needs to rebuild, it rebuilds the _whole_ data set. You're data from day 1 hasn't changed, but it has to be included in the data that gets refreshed. Gradually, this causes the model to rebuild more and more slowly, frustrating your end users who just want to know how many times people viewed the product page for your new special flying mattress yesterday.

With dbt, you can build models incrementally, only building or rebuilding the rows that have been created or changed since you last ran the model. This can substantially speed up the build of your models. It will also greatly reduce the load on your database when the models build, allowing other queries to be served more quickly. 

The **second** problem is a tendency to never build intermediary PDTs to speed up model builds.

There is a tendency in Looker to only build views (and therefore PDTs) that are necessary as a component of an explore. Often though, it would be beneficial for the speed of your rebuilds to break one long model into a handful of tables and then reference those in a final query. `base_pdt` -> `intermediary_pdt` -> `final_pdt` can often be quicker than if all the logic exists in one final larger table. It's not that this can't be done in Looker - it's just that it feel wrong to generate all those views in Looker if they aren't directly for analytics purposes	. 

The workflow described above is common place in dbt. Ultimately, it creates a separation between the preparation of data and the LookML that instructs Looker how to expose your final modelled output. 

## Testing your models

Another feature that dbt brings to the table is model testing, similar to unit tests present in most software development languages. 

Looker's data governance capabilities are fantastic. They ensure that everyone is looking at the same data - the same metrics. However, it doesn't ensure that the data feeding those metrics is correct. If someone makes a mistake in the SQL, everyone is still looking at the same metric, that metric just isn't accurate anymore. 

Imagine the following situation. You use Salesforce as your CRM and you have a LookML view called `sf_account`:

{{< highlight sql >}}

select 
	a.id,
	a.name
from salesforce.sf_account a

{{< /highlight >}}

Everything is working correctly. You're completely confident in the data generated in your reports.

Then, you decide you want to add the stage name from the related Salesforce opportunities to the model. Absent-mindedly, you forget that Accounts -> Opportunities is a one-to-many relationship. You make the following change and all your data from the `sf_account` view is suddently fanned out, distorting metrics in all your reports. 

{{< highlight sql >}}

select 
	a.id,
	a.name
from salesforce.sf_account a
left join salesforce.sf_opportunity o
	on o.accountid = a.id

{{< /highlight >}}

The ID column was supposed to be your unique primary key and now it isn't. There is currently no automated mechanism in Looker to test the output of that query. No way of knowing that a record for ID `176` is now there 5 times, one for each opportunity generated for that account.

In dbt, you can define constraints on the model, giving you the ability to identify instances like these in your development workflow.

{{< highlight yaml >}}

- name: sf_account
  columns:
    - name: id
      tests:
        - not_null
        - unique

{{< /highlight >}}

When you run `dbt test`, you'd realise your mistake and fix it before your Head of Sales starts shouting at you because all the numbers changed. Though testing of that sort hasn't always been common-place in analytics, it's incredibly important so that everyone can have confidence in the reporting they consume.

## PDT table names

David from your data science team wants to build a fancy model, predicting how many flying mattresses you'll sell next week. To do so, he wants to use the `customer_status` dimension defined in your `customers` view in Looker. He doesn't want to have re-define it in his work. He understandably thinks you should all be using the same definition. 

He jumps into his database explorer to find the name of the PDT in the `looker_scratch` schema, the location all the PDTs are built. He finds it. The table is called `lr$zzzxu3bfa1lkc2x5ne75_customers`. He goes off and builds his model. It has an AUC score of 0.96. He's incredibly pleased. He shares the analysis. 

Your COO opens it up, tries to run it. To his (and David's) dismay, the following error occurs: `ERROR: relation "looker_scratch.lr$zzzxu3bfa1lkc2x5ne75_customers" does not exist`. The table doesn't exist anymore. 

Every time you make a change to the SQL in a PDT, Looker will rename the underlying table in your database. As is the case here, Looker is often not the only platform or location where data analysis will be produced in your business. Often you'll have a data science function. Often you'll have people who need to write ad-hoc sql queries for specific pieces of analysis. As David initially asserted, all those functions should be building off the same models and the same definitions, building off the same single source of truth. 

The way Looker builds its PDTs isn't conducive to that principal. The table names will continue to change (and aren't memorable for analysts to use in ad-hoc querying). 

dbt will build your `customers` model in a table called `customers`. That will never change unless you tell it to. It can be consumed by every user and every platform in perpetuity. David can use it for his analysis. He could even put that model into production because he can be confident that your analytics team controls if that table exists or doesn't.

## Moving away from Looker (god forbid)

That leads me to a slightly meta point. 

Looker is great. I love Looker. I wholly endorse Looker. I don't see that changing any time soon. 

That said, eventually, you or I may choose to move to a different BI platform. It may no longer be the tool best suited for what you are trying to achieve.

At that point, given that the underlying data models need to be used by other users and platforms, you're going to need to retain those models somewhere. Might I suggest dbt?

![dbt business intelligence stack!](/dbt-stack-diagram.png)

## DRY code

SQL is a notoriously WET (write every time) language. It's veritably soaking with repetition. 

Need to pivot out a column into 5 columns? 5 lines of SQL.

Need to group by 76 columns? (Not sure why.) You need to write out every number between 1 and 76. 

Need to calculate the time between two dates, subtracting for weekends in between? The same lovely SQL snippet over and over and over.

Most people's SQL is rife with repetition. Most people's SQL is rife with repetition. 

This isn't a Looker problem. But it is a problem that frequently presents itself in PDT modelling. dbt's Jinja-powered 'macros' lets you write far more DRY (don't repeat yourself) code. It allows for more manageable, scalable SQL queries that your analysts will be happy to manage.

Want to pivot that `colour` column? Initially, you might have written something like this:

{{< highlight sql >}}

select
	id,
	sum(case when colour = 'Red' then amount end) as pivot_red,
	sum(case when colour = 'Yellow' then amount end) as pivot_yellow,
	sum(case when colour = 'Green' then amount end) as pivot_green
from colour_table
group by 1

{{< /highlight >}}

Instead, you can write a small macro and generate SQL as follows:

{{< highlight sql >}}

select
	id,
	{{ pivot('colour', ['Red','Yellow','Green']) }}
from colour_table
group by 1

{{< /highlight >}}






## When PDTs are best




A throwaway line I like but haven't fit anywhere:

dbt is [a hell of a drug](https://www.youtube.com/watch?v=udNHsk57f24), often discovered dbt through the powerful gateway drug of LookML persistent derived tables (PDTs).


