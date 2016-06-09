CDR rating for CCNQ
===================

    seem = require 'seem'

Data
----

* doc.account Store metadata about an account.
* doc.account.master The master / branding name for the account.
* doc.account.timezone Billing timezone
* doc.account.ratings{} for each `start-date` expressed as `YYYY-MM-DD`, provides a record
* doc.account.ratings{start-date}.table the name of the rating table

```json
{
 "_id": "account:{account}",
 "type": "account",
 "account": "{account}",

 "master": "{master}",
 "timezone": "{timezone}",
 "rating": {
  "{start-date}": {
   "table": "{tarif}"
  },
  "{start-date}": {
   "table": "{tarif}"
  }
 },
}
```

    class Rating

      constructor: (@cfg,@source,@PouchDB,@table_prefix = 'tarif') ->

      rate: seem (o) ->
        assert o.direction?
        assert o.from?
        assert o.to?
        assert o.stamp?
        assert o.duration?

        o.source = @source
        assert o.source_id?

        switch o.direction
          when 'ingress'
            o.billable_number = o.to
            o.remote_number = o.from
          when 'egress'
            o.billable_number = o.from
            o.remote_number = o.to
          else
            throw new Error "invalid direction: #{o.direction}"

Client-side data

        rate_client_or_carrier = seem (data,db_prefix) =>
          assert data.rating?
          assert data.timezone?

          rated = {}
          rated.db_prefix = db_prefix

          rated.timezone = data.timezone
          unless rated.timezone?
            throw new Error 'No timezone found'

          connect_stamp = moment.tz o.stamp, rated.timezone
          rated.connect_stamp = connect_stamp.format()

          rated.rating = rating_of data.rating, connect_stamp, o.timezone
          rated.rating_table = [@table_prefix,rated.rating.table].join '-'

          rated.period = @period_for rated
          rated._target_db = "rated-#{db_prefix}-#{rated.period}"

          rating_db_name = [@cfg.rating_tables,rated.rating_table].join ''
          rating_db = new @PouchDB rating_db_name

          try
            configuration = yield rating_db.get 'configuration'

            rating_data = yield find_prefix_in o.remote_number, rating_db

            if rating_data?.destination?
              rating_data = yield rating_db
                .get "destination:#{rating_data.destination}"
          finally
            close_pouch rating_db

          rating_db = null

          assert configuration?.currency?, "No currency in configuration of #{rating_db_name}"

          rated.currency = configuration.currency
          rated.divider = configuration.divider ? 1
          rated.per = configuration.per ? 60

          assert rating_data?, "No rating data for #{o.remote_number} in #{rating_db_name}"
          assert rating_data.description?, "No description for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.country?, "No country for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.initial?, "No initial for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.initial.cost?, "No initial cost for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.initial.duration?, "No initial duration for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.subsequent?, "No subsequent for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.subsequent.cost?, "No subsequent cost for #{rating_data._id} in #{rating_db_name}"
          assert rating_data.subsequent.duration?, "No subsequent duration for #{rating_data._id} in #{rating_db_name}"

          rated.description = rating_data.description
          rated.country = rating_data.country
          # etc.

          initial = rated.initial = rating_data.initial
          subsequent = rated.subsequent = rating_data.subsequent

assuming call was answered

          call_duration = o.duration
          if call_duration <= initial.duration
            amount = initial.cost
          else

periods of s.duration duration

            periods = Math.ceil (call_duration-initial.duration)/subsequent.duration
            amount = initial.cost + (subsequent.cost/configuration.per) * (periods*subsequent.duration)

round-up integer

          integer_amount = Math.ceil amount

this is the actual value (expressed in tarif.currency)

          actual_amount = integer_amount / configuration.divider

          rated.amount = amount
          rated.periods = periods
          rated.integer_amount = integer_amount
          rated.actual_amount = actual_amount

          rated

        if o.client?
          debug 'Processing client', o.client
          client_data = yield @cfg.prov.get "account:#{o.client}"

          o.master = client_data.master

          client_rated = yield rate_client_or_carrier client_data, "#{o.master}-#{o.client}"

        if o.carrier?
          debug 'Processing carrier', o.carrier
          carrier_data = yield @cfg.prov.get "carrier:#{o.carrier}"

          carrier_rated = yield rate_client_or_carrier carrier_data, o.carrier

Finalize record

        return [] unless client_rated? or carrier_rated?

        o._id = [
          o.billable_number
          client_rated?.connect_stamp ? carrier_rated?.connect_stamp
          o.remote_number
          o.duration
        ].join '-'

Prepare return value

        r = []

        push = (rated) ->
          return unless rated?
          w = {}
          for own k,v of o
            w[k] = v
          for own k,v of rated
            w[k] = v
          r.push w

        push client_rated
        push carrier_rated

The return values (in the array) should be stored in `_target_db` as `_id`.

        return r

      period_for: (side) ->
        moment
          .tz side.connect_stamp, side.timezone
          .format 'YYYY-MM'

      client_from_account: (account) ->
        return account

Rate a FreeSwitch CDR
---------------------

      rate_from_freeswitch: seem (cdr) ->
        vars = cdr.variables
        stamp = vars.answer_uepoch.replace /\d\d\d$/, ''

        @rate

Required

          direction: vars.ccnq_direction
          from: vars.ccnq_from_e164
          to: vars.ccnq_to_e164
          stamp: (moment stamp).format()
          duration: vars.billsec

          source_id: cdr._id

Required if present

          client: @client_from_account vars.ccnq_account
          carrier: vars.ccnq_carrier

Optional (defaults are provided)

          timezone: vars.ccnq_timezone

Unused

          connect_ms: stamp

    close_pouch = require './close_pouch'

    module.exports = Rating

    rating_of = require './lib/rating_of'
    find_prefix_in = require './lib/find_prefix_in'
    moment = require 'moment-timezone'
    assert = require 'assert'
    pkg = require './package'
    debug = (require 'debug') pkg.name
