Overview
========

We build a pipeline

    raw-cdr | rating | aggregation | invoicing

* `rating` translates CDR-by-CDR, applying two rating tables, one for clients, one for carriers;
* `aggregation` manages billing periods and applies contract elements (e.g. limits per billing period); only applied to clients;
* `invoicing` is out-of-scope.

For each call we create:
- a `trace` CDR, used for troubleshooting;
- two rated CDRs:
  - one for "client" invoicing;
  - one for "carrier" invoicing.
- an invoicing CDR.

Since LCR routing is now donne client-side, CDR generation will also happen client-side.


Project Plan
============

Rating
------

Generate rated CDRs from call data. [Projet entertaining-crib](https://gitlab.k-net.fr/shimaore/entertaining-crib)

Aggregation
-----------

Projet astonishing-competition

Mapping a CDR to a rating table
===============================

We add two fields, `timezone` and `rating`, in all `endpoint` records:

    {
      "type": "endpoint"

      "timezone": "{timezone}",
      "rating": {
        "{start-date}": {
            "table": "{tarif}"
            "forfait": "{forfait}"
        },
        "{start-date}": {
            "table": "{tarif}"
            "forfait": "{forfait}"
        }
      },

    }

- `{timezone}`: billing timezone
- `{start-date}`: `YYYY-MM-DD` (sorted alpha-numerically)
- `{tarif}`: used in rating
- `{forfait}`: used in aggregation


Rating tables
=============

Naming conventions
------------------

Each rating table is identified by its name. Example conventions might be:

- `{tarif}` = `{client|carrier}-{start}`
- `{client|carrier}` = `{client}` | `{carrier}`
- `{client}` = name of client tariff
- `{carrier}` = name of carrier tariff
- `{start}` = start of applicability (`YYYYMMDD`, ISO8601)

__Each rating table is created once and for all, and never modified.__

Each rating table is stored in a CouchDB database called `rates-{tarif}`.

Each rating table contains records as defined in the following sections.

Configuration
-------------

One single record per rating database:

    {
      "_id": "configuration"

    , "name": {
        "en-US": "Tariff unlimited-special, starting October 12, 2015
        "fr-FR": "Tarif illimité spécial, au 12 octobre 2015"
      }

    , "currency": "EUR"
    , "divider": 1000
    , "per": 60
    }


- `divider`: divider used on tariffs. In this example, values are computed in thousandth of Euros.
- `per`: duration used for tariffs, defaults to 60 meaning all prices are expressed as "per minute" (even though computations use the `time` parameters, for example "per second starting with the first second").
- `currency`: invoicing currency
- `name`: names of the rating table in different locales.

The `configuration` record is copied in the rated CDR in the field `configuration`.

Prefixes
--------

There are two ways to provide rating data for a given prefix:

1. Mapping prefix → destination, to send multiple prefixes into a common `destination` record (see below):

        {
          "_id": "prefix:336"
        , "type": "prefix"
        , "prefix": "336"

        , "destination": "fr-mobile"
        }

2. Storing rating data directly inside the prefix record, for example for individual numbers with dedicated costs:

        {
          "_id": "prefix:3303614"
        , "type": "prefix"
        , "prefix": "3303614"

        , "description": {
            "fr-FR": "RSVA 3614"
          }
        , "country": "fr"
        , "fixed": false
        , "mobile": false
        # etc. see https://gitlab.k-net.fr/shimaore/numbering-plans for appropriate fields

        , "initial": {
            "duration": 60
          , "cost": 2000
          }
        , "subsequent": {
            "duration": 10
            "cost": 345
          }
        }

    Notes: `duration` is expressed in seconds, `cost` is expressed in `currency`*`divider`.

    In this example, costs is 2€ at connection time for the first minute, plus 0.345€/minute billed by 10s increments.

Destination
-----------

The `destination` records are used to ease translation.

    {
      "_id": "destination:fr-mobile"
    , "type": "destination"
    , "destination": "fr-mobile"

    , "description": {
        "fr-FR": "Mobile France"
      }
    , "mobile": true
    , "country": "fr"

    , "initial": {
        "duration": 0
      , "cost": 0
      }
    , "subsequent": {
        "duration": 1
        "cost": 12
      }
    }

Notes: `duration` is expressed in seconds, `cost` is expressed in `currency`*`divider`.

In this example, there are no connection costs, cost are 0.012€/minute rated per second starting with the first second of the call.

What does the rating code do?
=============================

The code will output rated records, with the indicated rates applied onto the record.

The records are generated in the middleware of the `earthy-slave` project, which uses the code in `entertaining-crib` to build the CDRs.

Time of rating
--------------

Rating is based on the tarif applicable at the time the call is connected, in the selected timezone.

Rated amount
------------

Pseudo-code:

    # assuming call was answered
    call_duration = ...
    if call_duration <= initial.duration
      amount = initial.cost
    else
      # periods of s.duration duration
      periods = Math.ceil (call_duration-initial.duration)/subsequent.duration
      amount = initial.cost + (subsequent.cost/tarif.per) * (periods*subsequent.duration)

    # roundup to the higher integer
    integer_amount = Math.ceil amount

    # this is the actual value (expressed in tarif.currency)
    actual_amount = integer_amount / tarif.divider

Contents
--------

Format of a rated CDR:

    {
      "_id": "{billable-number}-{connect-stamp}-{remote-number}-{duration}"

      "source": "{CDR source table/DB}"
      "source_id": "{reference of the record in the source table/DB}"

      "rating": `rating` record for the date of the call
      "rating_table": full (CouchDB database) name of the rating table used
      "rating_data": {
        initial: {
          cost:
          duration:
        }
        subsequent: {
          cost:
          duration:
        }
        # and other data found in the `destination` record
      }

      "billable_number": CCNQ E.164 billable number (`33972222713`)
      "connect_stamp": ISO8601 connect stamp in local timezone
      "timezone":
      "remote_number":
      "duration":

      "period"

      "prefix" (object, from database)
      "destination" (object, from database, if applicable)
      "configuration" (object, from database)

      "periods"
      "amount"
      "integer_amount"
      "actual_amount"
    }

Target database
---------------

Note that the storing code is located in module `astonishing-competition`, not in the rating modules.

- `{period}` name is algorithmic; by default = `YYYY-MM` based on local time

Carrier-side: `rated-{carrier}-{period}`

Client-side: `rated-{client}-{period}`

Source code
-----------

Projects `entertaining-crib` (rating algorithm) and `earthy-slave` (FreeSwitch middleware).

What does the `aggregation` code
================================

The aggregation code in `astonishing-competition` uses the database-driven `flat-ornament` code execution module to convert a rated CDR into an invoicing CDR.

Source
------

Projet `astonishing-competition`.
