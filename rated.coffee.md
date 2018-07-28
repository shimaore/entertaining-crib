CDR rating for CCNQ
===================

    Base = require 'base-p'
    base62 = new Base 62

    Moment = require 'moment-timezone'

    module.exports = class Rated

The constructor will make a copy of the data.

      constructor: (data) ->
        for own k,v of data
          this[k] ?= v
        @type = 'rated'

`toJS` will return a copy of the data.

      toJS: ->
        Object.assign {}, this

      assert: (name) ->
        unless this[name]
          throw new Error "Missing #{name}", this

      compute: (duration,ignore = 0) ->

        @duration ?= duration if duration?

assuming call was answered (but no duration recorded yet)

        unless @duration?
          return

        if @duration < ignore
          @ignore = true
          return

        @assert 'billable_number'
        @assert 'connect_stamp'
        @assert 'remote_number'

        short = (t) -> if t.match /^\d+$/ then base62.encodeBig BigInt t else t

        @_id = [
          short @billable_number
          base62.encode Moment(@connect_stamp).valueOf()
          short @remote_number
          base62.encode @duration
        ].join '-'

        @assert 'rating_data'
        @assert 'per'
        @assert 'divider'

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
        return
