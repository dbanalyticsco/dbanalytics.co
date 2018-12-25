---
title: "Converting Looker PDTs to dbt models: Reasons why (and why not)"
date: "2018-12-25"
author: "Dylan Baker"
---

Over the past few years, Looker has become a market-leading BI tool. Its data governance and self-service capabilities have now become paramount to thousands of BI stacks. For many, the Third Wave of BI (TM Frank Bien) is well and truly here.

Persistent derived tables (PDTs) are a big part of Looker's success. They allow businesses to quickly transform their data with simple SQL `select` statement, abstracting lots of the data engineering that might otherwise be required. They allow you to easily materialize those models as tables in your data warehouse, speeding up the query times for your end users. 

They are addictive. Build a few PDTs, define an explore. Push. Build a few more. Push. Re-define the calculation behind a dimension. Push. Like most addictions though, there comes a point where you realise it may be detrimental to your health and well-being. Well, that of your BI stack anyway. 

Models can start getting slow, without the ability to optimise fully. Models can get repetitive, with very portions of code written out over and over. Models can break, without the ability to test their output.

### Enter: dbt.

dbt is to PDTs what Reeses is to peanut butter slathered on store-brand chocolate: broadly the same thing, but one far better executed that the other.

From Tristan Handy, the CEO and founder of Fishtown Analytics who build dbt:

> dbt is the T in ELT. It doesn’t extract or load data, but it’s extremely good at transforming data that’s already loaded into your warehouse. This “transform after load” architecture is becoming known as ELT (extract, load, transform).

> Every model is exactly one SELECT query, and this query defines the resulting data set.

Sounds pretty similar, right? It is. dbt simply allows you to abide by certain principals that PDTs don't. dbt allows you some flexibity that PDTs don't.

In this article, I'm going to cover some of the issues with doing all of the transformative modelling in Looker. I'll describe the pitfalls that may befall you and I'll lay out an argument for why dbt may be a better option for some of your modelling. In the next part of this series, I'll outline the practical steps of actually moving your PDTs into dbt.

### A quick note

Before I go any further, I want to remark that I wholeheartedly believe that Looker is a fantastic BI tool. It has been transformative for my work at a number of different businesses. It has powered most of the reporting I have done in the past three years. It is great. I fully intend to work with Looker for years to come. I wholly endorse it. 

It excels at data governance. It excels at self-service. It excels at integrating with other tools. It's just not the best tool for certain types of data modelling.

So, with that said, here are the instances where I would recommend a move to dbt and the instances where Looker is going to better serve your needs.






A throwaway line I like but haven't fit anywhere:

dbt is [a hell of a drug](https://www.youtube.com/watch?v=udNHsk57f24), often discovered dbt through the powerful gateway drug of LookML persistent derived tables (PDTs).



