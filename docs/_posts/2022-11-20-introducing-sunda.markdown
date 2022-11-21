---
layout: post
title: 'Using SQL like syntax to query JSON with Sunda'
date: 2022-11-21 18:20:00 +0000
categories: personal-projects sunda
---

# Introduction

Sunda is a project that I started about 2 years ago at the time of writing. It is a small tool written in TypeScript that allows for the querying of JSON arrays via an SQL-like language.

Who did I write this tool for? Me of course! In my day to day job I often find myself pouring through large JSON arrays returned by some kind of API (the AWS api being a common one), and I was often finding myself wishing I had some more powerful tooling to search for relevant data inside of the array.

A lot of this can be facilitated by `jq`, but there's 1 small issue, I don't personally _like_ `jq`'s query syntax. This isn't a dig at `jq`, it is a very powerful tool that I make frequent use of when building CI/CD pipelines and other automated scripts that handle JSON data. But when I am using a tool for some adhoc investigative work, I want something that feels more intuitive, and to me personally, `jq`'s query syntax just doesn't feel intuitive to write.

So, that begs the question, what _is_ an intuitive query syntax to write? Well for most devs, myself included, SQL is an excellent example of a query language that feels intuitive. Being able to use a SQL-like language to query random blobs of JSON data that have been pulled from an API or exported from the AWS console would be very, very useful.

So, with that logic in mind, I embarked on writing `sunda`, a small utility that could read arbitrary JSON data from `stdin` or a specified file, and then query arrays of data that reside in that JSON.

The source code of the project can be found [here](https://github.com/GeekFiftyFive/Sunda).

# High Level Details

Sunda is written in TypeScript, and has 0 production dependencies. The only dependencies are dev dependencies, e.g. TypeScript for transpiling the code and Jest for testing.

Releases of the project are hosted on `npm` and can be found [here](https://www.npmjs.com/package/sunda). Deploying releases of the project to `npm` means it can be installed and ran very easily by running `npx sunda` or `npm install -g sunda` followed by `sunda`.

# Basic Demo

For a complete and up to date rundown of all of Sunda's functionality, the docs can be found [here on GitHub](https://github.com/GeekFiftyFive/Sunda/blob/main/README.md) or on the [npm registry page](https://www.npmjs.com/package/sunda).

For the sake of keeping this page concise, I will be giving a short demo of the sort of functionality `sunda` supports at the time of writing.

I have hosted an example JSON file <a href="{{ site.url }}/assets/sunda-demo.json" target="_blank">here</a>. This data was generated using a simple script that utilizes [faker-js](https://www.npmjs.com/package/@faker-js/faker).

This file can be downloaded directly, or it can be piped straight from curl to sunda, so long as a query is passed in using either the `-q` or `--query` parameters. Passing one of these parameters takes `sunda` out of REPL mode and instead enables it to read data from `stdin`, and write it either to `stdout` or to an output file using the `-o` or `--output` parameters.

Let's run a simple query against the data in the file mentioned earlier! For starters, lets just check for string equality against one of the fields! Running the following query:

```sql
SELECT * FROM People WHERE firstName = 'Alexandre'
```

Nets us this result:

```json
[
  {
    "firstName": "Alexandre",
    "lastName": "Bahringer",
    "company": "Fay - Balistreri",
    "favoriteSongs": [
      "Money For Nothing",
      "Dance to the Music",
      "The Sign",
      "I'll Make Love to You"
    ],
    "favoriteGenre": "Blues",
    "age": 53,
    "recentRelease": "Dreamlover"
  }
]
```

Note, the result has been prettified for readability.

Say we didn't just want to match one string literally, and wanted to match on multiple different strings that all contain a certain pattern. Running the following query:

```sql
SELECT firstName, lastName FROM People WHERE firstName like '%li%'
```

This will match all entries where the `firstName` field contains the substring `li`, as well as only selecting the `firstName` and `lastName` fields, giving us the following result:

```json
[
  {
    "firstName": "Ophelia",
    "lastName": "Durgan"
  },
  {
    "firstName": "Aurelie",
    "lastName": "Bednar"
  },
  {
    "firstName": "Olin",
    "lastName": "Sanford"
  }
]
```

We can even pull out regex capture groups using a special function called `REGEX_GROUP`! If we run the following query:

```sql
SELECT REGEX_GROUP('^(\w*)li(\w*)$', firstName, 1) FROM People WHERE firstName like '%li%'
```

`sunda` will use the ECMAScript regex `/^(\w*)li(\w*)$/` against the `firstName` field and pull data out of the first capture group, netting us the following result:

```json
[
  {
    "0": "Ophe"
  },
  {
    "0": "Aure"
  },
  {
    "0": "O"
  }
]
```

`sunda` is also able to check the contents of arrays. The `ARRAY_POSITION` function will return the 1 base index of where an item exists in an array, or `null` if the item does not exist in the array.

Say we want to find all of the People in the list where the `favoriteSongs` field contains the song `Rock With You`. This can be achieved using the following query:

```sql
SELECT * FROM People WHERE ARRAY_POSITION(favoriteSongs, 'Rock With You') > 0
```

This will yield the following results:

```json
[
  {
    "firstName": "Meta",
    "lastName": "Gerlach",
    "company": "Pollich - Terry",
    "favoriteSongs": [
      "God Bless the Child",
      "The Way You Look Tonight",
      "Rock With You",
      "Battle of New Orleans"
    ],
    "favoriteGenre": "Non Music",
    "recentRelease": "Hey Paula",
    "age": 84
  },
  {
    "firstName": "Olin",
    "lastName": "Sanford",
    "company": "Nader and Sons",
    "favoriteSongs": [
      "Green River",
      "Kung Fu Fighting",
      "Rock With You",
      "I Will Follow Him"
    ],
    "favoriteGenre": "Rock",
    "age": 64,
    "recentRelease": "A Boy Named Sue"
  }
]
```

Finally, say we wanted to get a count of all of the unique values of the `favoriteGenre` field, we can use the `distinct` keyword wrapped inside of a `count`.

```sql
SELECT COUNT(DISTINCT favoriteGenre) FROM People
```

This will yield the following result:

```json
[{ "count": 11 }]
```

Note that sunda always wraps the output in an array.

# "Flexing" Its Capabilities

The above queries were all rather tame, and Sunda is capable of executing much more sophisticated queries than the ones demo'd above. As such, I've written one final query to demonstrate more of Sunda's capabilities.

Lets take this query:

```sql
SELECT SUM(songs.plays)
FROM   people
JOIN   songs
WHERE  People.recentRelease = Songs.songName
AND    ((Songs.plays + 1200) / People.age > 65)
AND    People.company LIKE '%er%'
OR     People.firstName = 'Adelle'
```

This demos some of Sunda's features that weren't demoed in the previous examples.

First of all, it is able to Join other fields in the input JSON so long as they also refer to an array.

It also demonstrates it's ability to perform arithmetic, and numeric comparisons, as well as being able to use the boolean operators `and` and `or`, with proper precedence.

Lastly it supports aggregations such as `count` and `sum`.

# Wrap Up

This blog post was just intended as a very brief high level overview of `sunda`. I would like to take a more in-depth look at how it actually works at a later date, but I have some fairly major refactors planned that I would like to finish first.

I hope this has been insightful, and that you may consider checking out the project for yourself!
