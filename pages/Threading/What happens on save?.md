# Intro to Threading in ServiceNow

Have you ever wondered what the difference is between the asynchronous technologies in ServiceNow? Well look no further, hopefully this article will help demistify the magic.

Let's start off with the most basic and probably well understood options -- Asynchronous Business Rules

## Async BR

Most are familiar with the standard business rules -- perhaps a "Before Insert" business rule that sets some default values on a record, or an "After Update" business rule that will run some quick calculations and write the results to a field.

But what about those times you need to run a **lot** of logic in response to a record change? Enter, the "Async" business rule. These business rules are great for when you don't need your logic to execute immediately, but rather _eventually_.

Let's bring this theory into reality -- take for example you have a table with a field called "Address". After the user saves this record, you desire to turn this address into a lat/lng. One way to do this is to transmit the data entered into the address field to an API that geocodes the address into a lat/lng. However, this round trip can take some time -- using a regular business rule and a synchronous REST call would mean the user has to sit and wait for this whole cycle to complete before the form refreshes after saving an address -- not a great user experience.

![](save_journey.svg)