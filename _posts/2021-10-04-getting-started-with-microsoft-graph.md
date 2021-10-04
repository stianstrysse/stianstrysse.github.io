---
layout: post
title: Getting started with Microsoft Graph
subtitle: Let's go from zero to somewhat hero by getting familiar with topics like REST API, JSON, HTTP methods, access tokens, permission scopes, Graph Exporer, Powershell SDK and more in this blogpost series covering Microsoft Graph.
categories: AZUREAD OFFICE365 GRAPH POWERSHELL
thumbnail-img: /assets/img/posts/2021-10-04/microsoft-graph-thumb.png
share-img: /assets/img/posts/2021-10-04/microsoft-graph-thumb.png
author: Stian A. Strysse
---

Intended for -- but not limited to -- IT Pros and developers who are familiar with Powershell or other scripting and code languages, who work with Microsoft cloud services, but still haven’t started to look into Microsoft Graph. The goal of this blogpost series is to understand what Microsoft Graph is and how to use it for administrative purposes and automation tasks in Azure AD plus related cloud services.

As you may be aware of, Microsoft is [deprecating and sunsetting the Azure AD Graph API on June 30, 2022](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/update-your-applications-to-use-microsoft-authentication-library/ba-p/1257363). Both the MSOnline (MSOL) and AzureAD Powershell modules utilize this API, which means that in some months from now these Powershell modules will likely no longer function.

Microsoft is asking people to start using [Microsoft Graph Powershell SDK module](https://docs.microsoft.com/en-us/powershell/microsoftgraph/overview?view=graph-powershell-beta) instead, which uses the Microsoft Graph API. We'll look at this Powershell SDK later in the blogpost series. 

Let's first start with the basics and gradually advance - skip the parts you're already familiar with.

**Part 1**:

+ [What is an API?](#what-is-an-api)
+ [What is a REST API?](#what-is-a-rest-api)
+ [What is JSON?](#what-is-json)
+ [How to fetch data from a REST API?](#how-to-fetch-data-from-a-rest-api)
+ [What is a HTTP status code?](#what-is-a-http-status-code)

**Part 2**:

+ [What is Microsoft Graph?](/blog/getting-started-with-microsoft-graph-part2/)
    + [API versions](/blog/getting-started-with-microsoft-graph-part2/#api-versions)
    + [Authentication](/blog/getting-started-with-microsoft-graph-part2/#authentication)
    + [Query parameters](/blog/getting-started-with-microsoft-graph-part2/#query-parameters)
    + [Request headers and body](/blog/getting-started-with-microsoft-graph-part2/#request-headers-and-body)
    + [Permission scopes and consent](/blog/getting-started-with-microsoft-graph-part2/#permission-scopes-and-consent)

**Part 3**:

+ [What is Graph Explorer?](/blog/getting-started-with-microsoft-graph-part3/)
    + [Azure AD testing environment](/blog/getting-started-with-microsoft-graph-part3/#azure-ad-testing-environment)
    + [Consenting to permission scopes in Graph Explorer](/blog/getting-started-with-microsoft-graph-part3/#consenting-to-permission-scopes-in-graph-explorer)
    + [Working with Graph Explorer](/blog/getting-started-with-microsoft-graph-part3/#working-with-graph-explorer)
    + [Request examples for user objects](/blog/getting-started-with-microsoft-graph-part3/#request-examples-for-user-objects)
    + [Request examples for group objects](/blog/getting-started-with-microsoft-graph-part3/#request-examples-for-group-objects)

**Part 4**:

+ [What is Microsoft Graph PowerShell SDK?](/blog/getting-started-with-microsoft-graph-part4/)
    + [SDK installation](/blog/getting-started-with-microsoft-graph-part4/#sdk-installation)
    + [SDK API version](/blog/getting-started-with-microsoft-graph-part4/#sdk-api-version)
    + [SDK authentication](/blog/getting-started-with-microsoft-graph-part4/#sdk-authentication)
    + [Working with the Powershell SDK](/blog/getting-started-with-microsoft-graph-part4/#working-with-the-powershell-sdk)
    + [SDK examples for user objects](/blog/getting-started-with-microsoft-graph-part4/#sdk-examples-for-user-objects)
    + [SDK examples for group objects](/blog/getting-started-with-microsoft-graph-part4/#sdk-examples-for-group-objects)

_

## What is an API?

API stands for **Application Programming Interface** and is simply put a service with a set of functions and procedures that allows applications to access and interact with data and services of other applications. An API makes it possible to create, read, update and delete data in a service. The API itself does not contain the data, but it governs and exposes the data to (usually authorized) applications which connects to the API. Most websites and apps on your smartphone today connects to one or more APIs when fetching data for you.

Two common API architectures are SOAP (Simple Object Access Protocol) which relies on XML, and REST (Representational State Transfer) which commonly uses JSON. Microsoft Graph a REST API.

## What is a REST API?

REST (also known as RESTful) API is the de facto standard for web services today with its lighter architecture and support for JSON. REST APIs are mostly used over HTTP with the following **verbs** as common methods:

- GET: read data
- POST: add new data
- PUT: update existing data
- PATCH: update a subset of data
- DELETE: remove data

To create, read, update or delete data (CRUD) in a REST API, a target is required - meaning the resource to do an action on. Resources in a REST API is often referred to as the **nouns** that the HTTP verbs act upon. 

| Action (Verb) | Resource (Noun) | Description                        |
| ------------- |:---------------:| ----------------------------------:|
| GET           | /api/users      | Get a list of users                |
| GET           | /api/users/id1  | Get a single user by ID            |
| POST          | /api/users      | Create a new user                  |
| PUT           | /api/users/id1  | Replace a single user by ID        |
| PATCH         | /api/users/id1  | Update data on a single user by ID |
| DELETE        | /api/users/id1  | Delete a single user by ID         |

REST APIs are usually consuming data and responding with data in JSON format, which is also true for Microsoft Graph.

## What is JSON?
JSON *(pronounced /ˈdʒeɪsən/;)* is short for JavaScript Object Notation. It is a format for storing and transmitting data objects containing key-value pairs and arrays. If you are familiar with Powershell, JSON shouldn’t be that hard to understand since Powershell too supports attribute-value pairs and arrays as it is an object oriented scripting language.

Fire up your favourite Powershell IDE (like [VSCode](https://code.visualstudio.com/docs/languages/powershell)) to have a closer look. With this Powershell code we construct a hashtable (key-value pairs) containing an array of email addresses:

```powershell
$array = @("first@learningbydoing.cloud","second@learningbydoing.cloud")

$hashtable = @{
    Name = "Dummy User"
    Country = "Norway"
    Account = @{
        Login = "dummy1"
        Emails = $array
    }
}
```

Type `$hashtable` and hit *Enter* in the console prompt to output the hashtable object:

```text
Name        Value
----        ----
Account     {Emails, Login}
Name        Dummy User
Country     Norway
```

Do the same with `$hashtable.Account` to output the array values:

```text
Name        Value
----        ----
Emails      {first@learningbydoing.cloud, second@learningbydoing.cloud}
Login       dummy1
```

Since Powershell supports JSON we can easily output this hashtable object as JSON with the command `$hashtable | ConvertTo-Json`:

```json
{
    "Account": {
            "Emails": [
                    "first@learningbydoing.cloud",
                    "second@learningbydoing.cloud"
             ],
             "Login":  "dummy1"
                },
    "Name": "Dummy User",
    "Country": "Norway"
} 
```

Now you know how a JSON object with an array looks like. Let’s try to fetch some data from an API.

## How to fetch data from a REST API?

There are many [public REST APIs](https://github.com/public-apis/public-apis) that you can test with. For this purpose, I'm using the **weather-api** with the following Powershell code:

```powershell
# Set city
$city = "New York"

# Fetch weather data from API for specified city
$request = Invoke-WebRequest -Method GET -Uri "https://goweather.herokuapp.com/weather/$city"

# Output results
$request | Select-Object StatusCode,StatusDescription,Content
```

The results will appear on the console:

```text
StatusCode  StatusDescription  Content
----------  -----------------  -------
       200  OK                 {"temperature":"+14 °C","wind":"0 km/h"... 
```

`$request.Content` contains the current weather in New York in JSON format. We can convert the JSON data to a PSCustomObject to work with it in Powershell by running `$json = $request.Content | ConvertFrom-Json`. Output `$json` to see the following results:

```text
temperature : +14 °C
wind        : 0 km/h
description : Partly cloudy
forecast    : {@{day=1; temperature=19 °C...
```

Note that you can make a REST-request in Powershell to get the results directly as PSCustomObject, which means you won't have to convert from JSON:

```powershell
Invoke-RestMethod -Method GET -Uri "https://goweather.herokuapp.com/weather/$city"
```

In the first result above we see that **StatusCode** was **200** and **StatusDescription** was **OK** – this is a HTTP status code and means that the API could process the request and give us the results successfully.

## What is a HTTP status code?

Rest APIs depend on HTTP standards, and whenever the REST API replies to a request it will send with a HTTP status code for communicating the result of the request – such as if the request was successful or not. The status code is communicated both with a machine-readable response and a human-readable description.

The most common HTTP status codes are:

- 200: Success
- 201: Created
- 401: Not authorized
- 403: Forbidden
- 404: Not found
- 429: Too many requests

As you can see, they are quite self-explaining. The good thing about these HTTP status codes is that you can easily verify them in a Powershell script to check if an API request was successful or not.

Alright, this concludes the very basics of REST APIs. Now let’s jump over to Microsoft Graph.

---

Go to **[Part 2: What is Microsoft Graph?](/blog/getting-started-with-microsoft-graph-part2/)**