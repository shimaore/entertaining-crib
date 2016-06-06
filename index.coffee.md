CDR rating for CCNQ
===================

Select the last rating period that encloses the call-stamp

    rating_of = (data,connect_stamp) ->
      rating_periods = Object
        .keys data.tarif
        .map (t) ->
          key: t
          value: moment.tz(t,o.timezone)
        .sort (a,b) ->
          if a.value.isBefore b
            return -1
          if a.value.isAfter b
            return 1
          return 0
        .reverse()

      rating_period = rating_periods.find (start_date) ->
        start_date.value.isSameOrBefore connect_stamp

      data[rating_period.key]

    find_prefix_in = seem (destination,database) ->

      ids = [0..destination.length]
        .map (l) -> "prefix:#{destination[0...l]}"
        .reverse()

      {rows} = database.allDocs
        keys: ids
        include_docs: true
      (row.doc for row in rows when row.doc?)[0]

    class Rating

      constructor: (@cfg) ->

      rate: seem (o) ->
        assert o.direction?
        assert o.from?
        assert o.to?
        assert o.stamp?
        assert o.duration?

        assert o.source?
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
          side = {}
          side.timezone = data.timezone
          unless side.timezone?
            throw new Error 'No timezone found'

          connect_stamp = moment.tz o.stamp, side.timezone
          side.connect_stamp = connect_stamp.format()

          side.rating = rating_of data, connect_stamp
          side.rating_table = "tarif-#{side.rating.tarif}"

          side.period = @period_for data
          side._target_db = "rated-#{db_prefix}-#{period}"

          rating_db = new PouchDB [@cfg.rating_tables,side.rating_table].join ''

          try
            configuration = yield db.get 'configuration'

            rating_data = yield find_prefix_in o.remote_number, rating_db

            if rating_data?.destination?
              rating_data = yield rating_db
                .get "destination:#{rating_data.destination}"
          finally
            close_pouch db

          assert configuration.currency?
          assert configuration.divider?
          assert configuration.per?

          side.currency = configuration.currency
          side.divider = configuration.divider
          side.per = configuration.per

          assert rating_data?

          side.description = rating_data.descriptiong
          side.country = rating_data.country
          # etc.

          initial = side.initial = rating_data.initial
          subsequent = side.subsequenet = rating_data.subsequent

          # assuming call was answered
          call_duration = o.duration
          if call_duration <= initial.duration
            amount = initial.cost
          else
            # periods of s.duration duration
            periods = Math.ceil (call_duration-initial.duration)/subsequent.duration
            amount = initial.cost + (subsequent.cost/tarif.per) * (periods*subsequent.duration)

          # round-up integer
          integer_amount = Math.ceil amount

          # this is the actual value (expressed in tarif.currency)
          actual_amount = integer_amount / tarif.divider

          side.amount = amount
          side.periods = periods
          side.integer_amount = integer_amount
          side.actual_amount = actual_amount

          side

        if o.client?
          client_data = yield @cfg.prov.get "account:#{o.client}"

          o.master = client_data.master

          client_rated = rate_client_or_carrier client_data, "#{o.master}-#{o.client}"

        if o.carrier?
          carrier_data = yield @cfg.prov.get "carrier:#{o.carrier}"

          carrier_rated = rate_client_or_carrier carrier_data, o.carrier

        o._id = [
          o.billable_number
          o.client.connect_stamp
          o.remote_number
          o.duration
        ].join '-'

        r = []

        push_this = (rated) ->
          return unless rated?
          w = {}
          for own k,v of o
            w[k] = v
          for own k,v of rated
            w[k] = v
          r.push w

        push client_rated
        push carrier_rated

        return r

      period_for: (side) ->
        moment
          .tz side.connect_stamp, side.timezone
          .format 'YYYY-MM'

      client_from_account: (account) ->
        return account

      rate_from_freeswitch: seem (db_name,cdr) ->
        vars = cdr.variables
        stamp = vars.answer_uepoch.replace /\d\d\d$/, ''

        @rate

Required

          direction: vars.ccnq_direction
          from: vars.ccnq_from_e164
          to: vars.ccnq_to_e164
          stamp: (moment stamp).format()
          duration: vars.billsec

          source: db_name
          source_id: cdr._id

Required if present

          client: @client_from_account vars.ccnq_account
          carrier: vars.ccnq_carrier

Optional (defaults are provided)

          timezone: vars.ccnq_timezone

Unused

          connect_ms: stamp
