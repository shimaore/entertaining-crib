CDR rating for CCNQ
===================

    seem = require 'seem'

Data
----

* doc.endpoint.timezone Billing timezone
* doc.endpoint.ratings{} for each `start-date` expressed as `YYYY-MM-DD`, provides a record
* doc.endpoint.ratings[start-date].table the name of the rating table
* doc.carrier.timezone Billing timezone
* doc.carrier.ratings{} for each `start-date` expressed as `YYYY-MM-DD`, provides a record
* doc.carrier.ratings[start-date].table the name of the rating table

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

    class Rated

      constructor: (data) ->
        for own k,v of data
          this[k] ?= v
        @type = 'rated'

      toJSON: ->
        w = {}
        for own k,v of this
          w[k] = v
        w

      compute: (duration,ignore = 0) ->

        @duration ?= duration if duration?

assuming call was answered

        return ignore unless @duration?

        if @duration < ignore
          @ignore = true
          return

        @_id = [
          @billable_number
          @connect_stamp
          @remote_number
          @duration
        ].join '-'

        initial = @initial = @rating_data.initial
        subsequent = @subsequent = @rating_data.subsequent

        call_duration = @duration
        if call_duration <= initial.duration
          amount = initial.cost
        else

periods of s.duration duration

          periods = Math.ceil (call_duration-initial.duration)/subsequent.duration
          amount = initial.cost + (subsequent.cost/@configuration.per) * (periods*subsequent.duration)

round-up integer

        integer_amount = Math.ceil amount

this is the actual value (expressed in configuration.currency)

        actual_amount = integer_amount / @configuration.divider

        @amount = amount
        @periods = periods
        @integer_amount = integer_amount
        @actual_amount = actual_amount

    class Rating

      constructor: (@cfg) ->
        @table_prefix = @cfg.table_prefix ? 'rates'
        @source = @cfg.source
        assert @source, 'Missing source'
        @PouchDB = @cfg.rating_tables
        assert @PouchDB, 'Missing rating_tables object'

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
            throw new Error "invalid direction: #{o.direction}"

        rate_client_or_carrier = seem (data) =>
          assert data.rating?, 'Internal error: missing `data.rating`'
          assert data.timezone?, 'Internal error: missing `data.timezone`'

          rated = new Rated o

          rated.duration = o.duration

          rated.timezone = data.timezone
          unless rated.timezone?
            throw new Error 'No timezone found'

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

            if rating_data?.destination?
              rated.destination = rating_data.destination
              rating_data = yield rating_db
                .get "destination:#{rating_data.destination}"
          catch error
            debug "rate_client_or_carrier: #{error.stack ? error}"
          finally
            close_pouch rating_db

          rating_db = null

          assert configuration?.currency?, "No currency in configuration of #{rating_db_name}"

          rated.configuration = configuration
          rated.currency = configuration.currency
          rated.divider = configuration.divider ? 1
          rated.per = configuration.per ? 60

          assert rating_data?, "No rating data for #{o.remote_number} in #{rating_db_name}"
          assert rating_data.initial?, "No `initial` for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.initial.cost?, "No `initial.cost` for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.initial.duration?, "No `initial.duration` for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.subsequent?, "No `subsequent` for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.subsequent.cost?, "No `subsequent.cost` for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.subsequent.duration?, "No `subsequent.duration` for #{rating_data._id} in #{rating_db_name}"

          rated.rating_data = rating_data

          rated.compute()

          rated

Main handling

        if o.client?
          debug 'Processing client', o.client

          client_rated = yield rate_client_or_carrier o.client
            .catch (error) ->
              debug "Client rating error: #{error.stack ? error}"
              null
          client_rated?.side = 'client'

        if o.carrier?
          debug 'Processing carrier', o.carrier

          carrier_rated = yield rate_client_or_carrier o.carrier
            .catch (error) ->
              debug "Carrier rating error: #{error.stack ? error}"
              null
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
    assert = require 'assert'
    pkg = require './package'
    debug = (require 'debug') "#{pkg.name}:rating"
