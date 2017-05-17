CDR rating for CCNQ
===================

    seem = require 'seem'

    report = (text,values...) ->
      debug "Error: #{text}", values...

    check = (ok,args...) ->
      return true if ok
      report args...

Data
----

* doc.src_endpoint.timezone Billing timezone
* doc.src_endpoint.rating{} for each `start-date` expressed as `YYYY-MM-DD`, provides a record
* doc.src_endpoint.rating[start-date].table the name of the rating table
* doc.carrier.timezone Billing timezone
* doc.carrier.rating{} for each `start-date` expressed as `YYYY-MM-DD`, provides a record
* doc.carrier.rating[start-date].table the name of the rating table

```json
{
 "timezone": "{timezone}",
 "rating": {
  "{start-date}": {
   "table": "{rates}"
  },
  "{start-date}": {
   "table": "{rates}"
  }
 },
}
```

    Rated = require './rated'

    class Rating

* cfg.table_prefix (string) The prefix used to build the names of rating tables. Default: `rates`
* cfg.rating_tables ignore
* cfg.source ignore

      constructor: (@cfg) ->
        @table_prefix = @cfg.table_prefix ? 'rates'
        @source = @cfg.source
        @PouchDB = @cfg.rating_tables
        unless @source?
          report 'Missing source'
        unless @PouchDB?
          report 'Missing rating_tables object'

      rate: seem (o) ->
        debug 'rate', o
        return unless o.direction?
        return unless o.from?
        return unless o.to?
        return unless o.stamp?

        o.source = @source
        return unless o.source?

        switch o.direction
          when 'ingress'
            o.billable_number = o.to
            o.remote_number = o.from
          when 'egress'
            o.billable_number = o.from
            o.remote_number = o.to
          else
            report 'Invalid direction', o.direction
            return

        rate_client_or_carrier = seem (data) =>
          return null unless check data.rating?, 'Missing `data.rating`', data
          return null unless check data.timezone?, 'Missing `data.timezone`'

          rated = new Rated o

          rated.duration = o.duration

          rated.timezone = data.timezone
          return null unless check rated.timezone?, 'Missing `data.timezone`', data

          connect_stamp = moment.tz o.stamp, rated.timezone
          rated.connect_stamp = connect_stamp.format()

          rated.rating = rating_of data.rating, connect_stamp, data.timezone
          rated.rating_table = [@table_prefix,rated.rating.table].join '-'

          rating_db_name = rated.rating_table
          debug "rate_client_or_carrier: using rating_db_name #{rating_db_name}"
          rating_db = new @PouchDB rating_db_name

          try
            configuration = yield rating_db.get 'configuration'

            rating_data = yield find_prefix_in o.remote_number, rating_db
            rated.prefix = rating_data.prefix

* doc.prefix.destination (rating)(string) The doc.destination used for rating.

            if rating_data?.destination?
              rated.destination = rating_data.destination
              rating_data = yield rating_db
                .get "destination:#{rating_data.destination}"
          catch error
            report "rate_client_or_carrier: #{error.stack ? error}"
          finally
            close_pouch rating_db

          rating_db = null

          return null unless check configuration?.currency?, "No currency in configuration of #{rating_db_name}"

          return null unless check configuration?.ready, "Table #{rating_db_name} is not ready."

* doc.configuration (rating)
* doc.configuration.currency (rating)(string) Currency for this rating db.
* doc.configuration.divider (rating)(integer) Divider for values in the `cost` fields. Default: 1, meaning the values are used as-is.
* doc.configuration.per (rating)(integer) Number of seconds `cost` fields are expressed for. Default: 60, meaning the rates are expressed per-minute.
* doc.configuration.ready (rating)(boolean) If `false`, the rating table may be edited, but is not ready yet for use in production as a tariff; if `true`, the rating table may no longer be edited, but is useable in production as a tariff.

          rated.configuration = configuration
          rated.currency = configuration.currency
          rated.divider = configuration.divider ? 1
          rated.per = configuration.per ? 60

* doc.prefix.initial.cost (rating)(integer) Cost for the initial call connexion, expressed for a doc.configuration.per seconds period, multiplied by doc.configuration.divider.
* doc.destination.initial.cost (rating)(integer) Cost for the initial call connexion, expressed for a doc.configuration.per seconds period, multiplied by doc.configuration.divider.
* doc.prefix.initial.duration (rating)(integer) Duration of the initial call connexion period.
* doc.destination.initial.duration (rating)(integer) Duration of the initial call connexion period.
* doc.prefix.subsequent.cost (rating)(integer) Cost for in-call time intervals, expressed for a doc.configuration.per seconds period, multiplied by doc.configuration.divider.
* doc.destination.subsequent.cost (rating)(integer) Cost for in-call time intervals, expressed for a doc.configuration.per seconds period, multiplied by doc.configuration.divider.
* doc.prefix.subsequent.duration (rating)(integer) Duration of in-call time intervals.
* doc.destination.subsequent.duration (rating)(integer) Duration of in-call time intervals.

          return null unless check rating_data?, "No rating data for #{o.remote_number} in #{rating_db_name}"
          return null unless check rating_data.initial?, "No `initial` for #{rating_data._id} in #{rating_db_name}"
          return null unless check rating_data.initial.cost?, "No `initial.cost` for #{rating_data._id} in #{rating_db_name}"
          return null unless check rating_data.initial.duration?, "No `initial.duration` for #{rating_data._id} in #{rating_db_name}"
          return null unless check rating_data.subsequent?, "No `subsequent` for #{rating_data._id} in #{rating_db_name}"
          return null unless check rating_data.subsequent.cost?, "No `subsequent.cost` for #{rating_data._id} in #{rating_db_name}"
          return null unless check rating_data.subsequent.duration?, "No `subsequent.duration` for #{rating_data._id} in #{rating_db_name}"

          rated.rating_data = rating_data

          rated.compute()

          rated

Main handling

        if o.client?
          debug 'Processing client', o.client

          client_rated = yield rate_client_or_carrier o.client
          client_rated?.side = 'client'

        if o.carrier?
          debug 'Processing carrier', o.carrier

          carrier_rated = yield rate_client_or_carrier o.carrier
          carrier_rated?.side = 'carrier'

Prepare return value

        r = {}
        r.client = client_rated if client_rated?
        r.carrier = carrier_rated if carrier_rated?
        r

Toolbox
=======

    close_pouch = require './close_pouch'

    module.exports = Rating

    rating_of = require './lib/rating_of'
    find_prefix_in = require './lib/find_prefix_in'
    moment = require 'moment-timezone'
    pkg = require './package'
    debug = (require 'tangible') "#{pkg.name}:rating"
