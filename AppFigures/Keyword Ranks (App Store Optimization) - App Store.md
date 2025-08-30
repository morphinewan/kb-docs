---
title: "Keyword Ranks (App Store Optimization) \| App Store API · Appfigures"
url: "https\://docs.appfigures.com/api/reference/v2/aso-keyword-ranks"
description: "\<null\>"
author: "\<null\>"
site_name: "\<null\>"
language: "\<null\>"
image: "\<null\>"
generated_at: "2025-08-30T01:39:57.774Z"
---

Keyword Ranks (App Store Optimization)
======================================

The `/aso` resource provides access to keyword rank trends and stats for keywords and apps tracked in your Appfigures account.

Getting Keyword Stats for an App
--------------------------------

### Request

GET /aso/stats_?{filters}_

### Arguments

Field

Type

Description

products

Product ID

The Appfigures ID of the app to get data for. At this time only one app can be set per request.

countries

String

the ISO code of the country to fetch results in. At this time only one country can be set per request.

device

Enum

The device to fetch data from. Only applicable for iOS apps. options: \`handheld\` and \`tablet\`.

start

Date

The date and time of the beginning of the date range to fetch.

end

Date

The date and time of the end of the date range to fetch.

granularity

Enum

\`daily\` or \`hourly\`. Some plans do not have access to hourly granularity.

### Response

Field

Type

Description

avg\_position

Number

The average of all ranks using the latest rank positions and not including unranked keywords.

avg\_position\_delta

Number

The average of all rank changes using the latest rank change and not including unranked keywords.

highest

Number

The highest position the app was seen in for any of the tracked keywords. The best value is 1.

highest\_delta

Number

The biggest increase (improvement) in rank using the latest position and latest change.

lowest

Number

The lowest position the app was seen in for any of the tracked keywords.

lowest\_delta

Number

The biggest decrease (decline) in rank using the latest position and latest change.

top\_5

Number

The number of keywords, from your tracked list, that have a rank greater than or equal to 5 right now.

top\_25

Number

The number of keywords, from your tracked list, that have a rank greater than or equal to 25 right now.

top\_100

Number

The number of keywords, from your tracked list, that have a rank greater than or equal to 100 right now.

ranked\_keywords

Number

The number of keywords, from your tracked list, that the selected app is ranked in.

total\_keywords

Number

The number of keywords in your tracked list.

positive

Number

The number of keywords, from your tracked list, where the app’s rank moved up (improved) during the selected date range.

negative

Number

The number of keywords, from your tracked list, where the app’s rank moved down (declined) during the selected date range.

unranked\_keywords

Number

The number of keywords, from your tracked list, that the selected app is not ranked in.

unchanged

Number

The number of keywords, from your tracked list, that have not had any rank changes in the selected date range.

### Example Response

// GET /aso/stats?
                products=333918141896&
                countries=US&
                granularity=hourly&
                start=2021-02-03T00:00:00&
                end=2021-02-09T17:00:00&
                device=handheld

{
  "avg\_position": 21,
  "avg\_position\_delta": 9,
  "highest": 1,
  "highest\_delta": 0,
  "lowest": 43,
  "lowest\_delta": 15,
  "top\_5": 1,
  "top\_25": 2,
  "top\_100": 2,
  "ranked\_keywords": 5,
  "total\_keywords": 5,
  "positive": 4,
  "negative": 0,
  "unranked\_keywords": 0,
  "unchanged": 1
}

Getting Rank Trends
-------------------

### Request

GET /aso_?{filters}_

### Arguments

Field

Type

Description

group\_by

Enum

How to group the results. At this time only the \`keyword\` value is supported.

products

Product ID

The Appfigures ID of the app to get data for. At this time only one app can be set per request.

countries

String

the ISO code of the country to fetch results in. At this time only one country can be set per request.

device

Enum

The device to fetch data from. Only applicable for iOS apps. options: \`handheld\` and \`tablet\`.

start\_date

Date

The date and time of the beginning of the date range to fetch.

end\_date

Date

The date and time of the end of the date range to fetch.

normalize

Bool

Whether the results should include entries for days without data or not.

granularity

Enum

\`daily\` or \`hourly\`. Some plans do not have access to hourly granularity.

page

Number

The page you’d like to fetch.

### Response

The response is paged, which means it’ll return two main sections: `metadata` and `results`

**Metadata.Resultset:**

Field

Type

Description

count

Number

The max number of results per page.

page

Number

The current page of results being returned.

total\_count

Number

The number of results for this request.

total\_pages

Number

The number of pages for this request.

**Results:**

Field

Type

Description

position

Number

The rank for the selected app for this search.

delta

Number

The change in rank, represented by the number of positions moved up or down, between the latest scan and the one before it, using hourly granularity.

last\_change

Date

The date of the last change in rank.

num\_apps

Number

The total number of apps that rank for this keyword.

keyword\_id

String

The keyword’s Appfigures ID.

keyword\_term

String

The search term. This field may include emoji.

popularity

Number

The popularity score for this keyword, on a scale of 1 – 100. This metric may not be available to all plans.

competitiveness

Number

The competitiveness score for this keyword, on a scale of 1 – 100. This metric may not be available to all plans.

### Example Response

// GET /aso?
          group\_by=keyword&
          products=333918141896&
          countries=US&
          device=handheld&
          start\_date=2021-02-03T00:00:00&
          end\_date=2021-02-09T17:00:00&
          normalize=true&
          granularity=hourly&
          time\_zone=utc&
          page=1

{
  "metadata": {
    "resultset": {
      "count": 500,
      "page": 1,
      "total\_count": 5,
      "total\_pages": 1
    }
  },
  "results": \[
    {
      "position": 12,
      "delta": 1,
      "last\_change": "2021-02-09T16:00:00",
      "num\_apps": 269,
      "keyword\_id": "1d86c5c8b1cdcae0ad30574e31e6288a",
      "keyword\_term": "stream tv",
      "popularity": 32,
      "competitiveness": 92,
      "importance": "6.12"
    },
    {
      "position": 29,
      "delta": 8,
      "last\_change": "2021-02-07T22:00:00",
      "num\_apps": 231,
      "keyword\_id": "456f3e168ea7f915cd5898bce0162ab4",
      "keyword\_term": "stream shows",
      "popularity": 5,
      "competitiveness": 94,
      "importance": "0.74"
    },
    {
      "position": 1,
      "delta": 0,
      "num\_apps": 252,
      "keyword\_id": "54818b05d116eadc7f67517a3a6e4b33",
      "keyword\_term": "discovery",
      "popularity": 52,
      "competitiveness": 100,
      "importance": "52.00"
    },
    {
      "position": 23,
      "delta": -1,
      "last\_change": "2021-02-09T09:00:00",
      "num\_apps": 241,
      "keyword\_id": "7d520907f526f36930df8491326b1dbc",
      "keyword\_term": "tv shows",
      "popularity": 39,
      "competitiveness": 100,
      "importance": "6.16"
    },
    {
      "position": 43,
      "delta": -1,
      "last\_change": "2021-02-09T16:00:00",
      "num\_apps": 273,
      "keyword\_id": "f7b44cfafd5c52223d5498196c8a2e7b",
      "keyword\_term": "stream",
      "popularity": 57,
      "competitiveness": 98,
      "importance": "7.70"
    }
  \]
}