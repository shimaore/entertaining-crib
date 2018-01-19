CDR rating for CCNQ
===================

    module.exports = class Rated

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
          amount = initial.cost + (subsequent.cost/@per) * (periods*subsequent.duration)

round-up integer

        integer_amount = Math.ceil amount

this is the actual value (expressed in configuration.currency)

        actual_amount = integer_amount / @divider

        @amount = amount
        @periods = periods
        @integer_amount = integer_amount
        @actual_amount = actual_amount
