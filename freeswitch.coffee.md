Rate a FreeSwitch CDR
---------------------

    Rating = require './rating'
    seem = require 'seem'

    class FreeSwitchRating extends Rating

      rate: (cdr) ->
        vars = cdr.variables

        if not vars?
          return super cdr

        # winner = JSON.parse vars.ccnq_winner ? {}
        attrs = JSON.parse vars.ccnq_attrs ? {}

        stamp = vars.answer_uepoch.replace /\d\d\d$/, ''

        data =

Required

          direction: vars.ccnq_direction
          from: vars.ccnq_from_e164
          to: vars.ccnq_to_e164
          stamp: (moment stamp).format()
          duration: vars.billsec

          source_id: cdr._id

        @rate_freeswitch data, vars

      rate_freeswitch: seem (data,vars) ->

Required if present

Note: `ccnq_carrier` was added in tough-rate 14.5.0
Note: `ccnq_endpoint` and `ccnq_endpoint_json` were added in huge-play 12.3.0

Data for the client

        switch
          when vars.ccnq_endpoint_json?
            data.client = JSON.parse vars.ccnq_endpoint_json
          when vars.ccnq_endpoint?
            data.client = yield @cfg.prov.get "endpoint:#{vars.ccnq_endpoint}"

Data for the carrier

        switch
          when vars.ccnq_carrier_json?
            data.carrier = JSON.parse vars.ccnq_carrier_json
          when vars.ccnq_carrier?
            data.carrier = yield @cfg.prov.get "carrier:#{vars.ccnq_carrier}"

Timezone (used to evaluate)

        switch
          when vars.ccnq_timezone?
            data.timezone = vars.ccnq_timezone
          when data.client?.timezone?
            data.timezone = data.client.timezone

Do not rate / report emergency calls to the client. (The )

        delete data.client if attrs.emergency

        Rating::rate.call this, data

Toolbox
=======

    module.exports = FreeSwitchRating

    moment = require 'moment-timezone'
    pkg = require './package'
    debug = (require 'debug') "#{pkg.name}:rating-freeswitch"
