Select the last rating period that encloses the call-stamp

    moment = require 'moment-timezone'

    rating_of = (ratings,connect_stamp,timezone) ->
      rating_periods = Object
        .keys ratings
        .map (t) ->
          key: t
          value: moment.tz(t,timezone)
        .sort (a,b) ->
          if a.value.isBefore b
            return -1
          if a.value.isAfter b
            return 1
          return 0
        .reverse()

      rating_period = rating_periods.find (start_date) ->
        start_date.value.isSameOrBefore connect_stamp

      if rating_period?
        ratings[rating_period.key]
      else
        null

    module.exports = rating_of
