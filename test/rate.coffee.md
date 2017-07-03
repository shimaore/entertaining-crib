    seem = require 'seem'
    pkg = require '../package'
    debug = (require 'tangible') "#{pkg.name}:test:rate"
    {expect} = chai = require 'chai'
    chai.should()
    PouchDB = (require 'pouchdb').defaults db: require 'memdown'

    describe 'Rating', ->
      Rating = require '../rating'
      rating_tables = (name) -> PouchDB name

      it 'should not rate unless client or carrier are specified', seem ->
        r = new Rating {rating_tables,source:'test1'}

        a = yield r.rate
          direction: 'egress'
          from: '33972222713'
          to: '18002255288'
          stamp: '2013-05-14 12:52:23'
          duration: 23
          source_id: 'd895b2ea-58fe-4215-aae7-ca66a6c3099c'

        Object.keys(a).should.have.length 0

      it 'should rate carrier', seem ->
        r = new Rating {rating_tables,source:'test2'}
        cheap = rating_tables 'rates-cheap'
        yield cheap.put
          _id:'configuration'
          currency: 'EUR'
          divider: 1000
          per: 60
          ready: true
        yield cheap.put
          _id:'prefix:1800'
          country: 'us'
          description: {
            'en-US': 'US Toll-Free'
          }
          initial:
            cost: 0
            duration: 0
          subsequent:
            cost: 34
            duration: 1

        a = yield r.rate
          direction: 'egress'
          from: '33972222713'
          to: '18002255288'
          stamp: '2013-05-14 12:52:23'
          duration: 23
          source_id: 'd895b2ea-58fe-4215-aae7-ca66a6c3099c'
          carrier:
            _id: 'carrier:default'
            timezone: 'America/New_York'
            rating:
              '2012-12-14': table:'cheap'

        Object.keys(a).should.have.length 1
        a.should.have.property 'carrier'
        a.should.not.have.property 'client'
        a.carrier.should.have.property 'billable_number', '33972222713'
        a.carrier.should.have.property 'remote_number', '18002255288'
        a.carrier.should.have.property 'source', 'test2'
        a.carrier.should.have.property 'source_id'
        a.carrier.should.have.property 'rating'
        a.carrier.should.have.property 'rating_table'
        a.carrier.should.have.property 'connect_stamp'
        a.carrier.should.have.property 'timezone'
        a.carrier.should.have.property 'duration'
        a.carrier.should.have.property 'carrier'
        a.carrier.should.have.property 'prefix'
        a.carrier.should.have.property 'configuration'
        a.carrier.should.have.property 'currency', 'EUR'
        a.carrier.should.have.property 'periods', 23
        a.carrier.should.have.property 'integer_amount', 14
        a.carrier.should.have.property 'amount'
        a.carrier.should.have.property 'actual_amount'
        a.carrier.should.have.property 'divider', 1000
        a.carrier.should.have.property 'subsequent'
        a.carrier.subsequent.should.have.property 'cost', 34
        a.carrier.should.have.property '_id'

      it 'should rate client', seem ->
        r = new Rating {rating_tables,source:'test3'}
        expensive = rating_tables 'rates-expensive'
        yield expensive.put
          _id:'configuration'
          currency: 'EUR'
          divider: 100
          per: 60
          ready: true
        yield expensive.put
          _id:'prefix:1900'
          country: 'us'
          description: {
            'en-US': 'US Special'
          }
          initial:
            cost: 0
            duration: 0
          subsequent:
            cost: 1500
            duration: 30

        a = yield r.rate
          direction: 'egress'
          from: '33972222713'
          to: '19005551212'
          stamp: '2013-05-14 12:52:23'
          duration: 23
          source_id: 'd895b2ea-58fe-4215-aae7-ca66a6c3099c'
          client:
            timezone: 'America/New_York'
            rating:
              '2012-12-14': table:'expensive'

        Object.keys(a).should.have.length 1
        a.should.have.property 'client'
        a.should.not.have.property 'carrier'
        a.client.should.have.property 'billable_number', '33972222713'
        a.client.should.have.property 'remote_number', '19005551212'
        a.client.should.have.property 'source', 'test3'
        a.client.should.have.property 'source_id'
        a.client.should.have.property 'rating'
        a.client.should.have.property 'rating_table'
        a.client.should.have.property 'connect_stamp'
        a.client.should.have.property 'timezone'
        a.client.should.have.property 'duration'
        a.client.should.have.property 'client'
        a.client.should.have.property 'prefix'
        a.client.should.have.property 'configuration'
        a.client.should.have.property 'currency', 'EUR'
        a.client.should.have.property 'periods', 1
        a.client.should.have.property 'integer_amount', 750
        a.client.should.have.property 'amount'
        a.client.should.have.property 'actual_amount'
        a.client.should.have.property 'divider', 100
        a.client.should.have.property 'subsequent'
        a.client.subsequent.should.have.property 'cost', 1500
        a.client.should.have.property '_id'

      it 'should rate client and carrier', seem ->
        r = new Rating {rating_tables,source:'test4'}
        expensive = rating_tables 'rates-client-2'
        yield expensive.put
          _id:'configuration'
          currency: 'EUR'
          divider: 100
          per: 60
          ready: true
        yield expensive.put
          _id:'prefix:1900'
          country: 'us'
          description: {
            'en-US': 'US Special'
          }
          initial:
            cost: 0
            duration: 0
          subsequent:
            cost: 1500
            duration: 30
        cheap = rating_tables 'rates-carrier-A'
        yield cheap.put
          _id:'configuration'
          currency: 'EUR'
          divider: 1000
          per: 60
          ready: true
        yield cheap.put
          _id:'prefix:1900'
          country: 'us'
          description: {
            'en-US': 'US Special'
          }
          initial:
            cost: 0
            duration: 0
          subsequent:
            cost: 340
            duration: 1


        a = yield r.rate
          direction: 'egress'
          from: '33972222713'
          to: '19005551212'
          stamp: '2013-05-14 12:52:23'
          duration: 23
          source_id: 'd895b2ea-58fe-4215-aae7-ca66a6c3099c'
          client:
            timezone: 'America/New_York'
            rating:
              '2012-12-14': table:'client-2'
          carrier:
            timezone: 'America/New_York'
            rating:
              '2012-12-14': table:'carrier-A'

        Object.keys(a).should.have.length 2
        a.should.have.property 'client'
        a.client.should.have.property 'billable_number', '33972222713'
        a.client.should.have.property 'remote_number', '19005551212'
        a.client.should.have.property 'source', 'test4'
        a.client.should.have.property 'currency', 'EUR'
        a.client.should.have.property 'periods', 1
        a.client.should.have.property 'integer_amount', 750
        a.client.should.have.property 'divider', 100
        a.client.should.have.property 'subsequent'
        a.client.subsequent.should.have.property 'cost', 1500
        a.client.should.have.property '_id'
        a.should.have.property 'carrier'
        a.carrier.should.have.property 'billable_number', '33972222713'
        a.carrier.should.have.property 'remote_number', '19005551212'
        a.carrier.should.have.property 'source', 'test4'
        a.carrier.should.have.property 'currency', 'EUR'
        a.carrier.should.have.property 'periods', 23
        a.carrier.should.have.property 'integer_amount', 131
        a.carrier.should.have.property 'divider', 1000
        a.carrier.should.have.property 'subsequent'
        a.carrier.subsequent.should.have.property 'cost', 340
        a.carrier.should.have.property '_id', '33972222713-2013-05-14T12:52:23-04:00-19005551212-23'

      it 'should rate destinations', seem ->
        r = new Rating {rating_tables,source:'test5'}
        cheap = rating_tables 'rates-cheap-5'
        yield cheap.put
          _id:'configuration'
          currency: 'EUR'
          divider: 1000
          per: 60
          ready: true
        yield cheap.put
          _id:'prefix:1800'
          destination: 'us-tollfree'
        yield cheap.put
          _id:'destination:us-tollfree'
          destination:'us-tollfree'
          country: 'us'
          description: {
            'en-US': 'US Toll-Free'
          }
          initial:
            cost: 0
            duration: 0
          subsequent:
            cost: 34
            duration: 1

        a = yield r.rate
          direction: 'egress'
          from: '33972222713'
          to: '18002255288'
          stamp: '2013-05-14 12:52:23'
          duration: 23
          source_id: 'd895b2ea-58fe-4215-aae7-ca66a6c3099c'
          carrier:
            timezone: 'America/New_York'
            rating:
              '2012-12-14': table:'cheap-5'
        .catch (error) -> debug "rate #{error.stack ? error}"

        Object.keys(a).should.have.length 1
        a.should.have.property 'carrier'
        a.should.not.have.property 'client'
        a.carrier.should.have.property 'billable_number', '33972222713'
        a.carrier.should.have.property 'remote_number', '18002255288'
        a.carrier.should.have.property 'source', 'test5'
        a.carrier.should.have.property 'source_id'
        a.carrier.should.have.property 'rating'
        a.carrier.should.have.property 'rating_table'
        a.carrier.should.have.property 'connect_stamp'
        a.carrier.should.have.property 'timezone'
        a.carrier.should.have.property 'duration'
        a.carrier.should.have.property 'carrier'
        a.carrier.should.have.property 'prefix'
        a.carrier.should.have.property 'configuration'
        a.carrier.should.have.property 'destination', 'us-tollfree'
        a.carrier.should.have.property 'currency', 'EUR'
        a.carrier.should.have.property 'periods', 23
        a.carrier.should.have.property 'integer_amount', 14
        a.carrier.should.have.property 'amount'
        a.carrier.should.have.property 'actual_amount'
        a.carrier.should.have.property 'divider', 1000
        a.carrier.should.have.property 'subsequent'
        a.carrier.subsequent.should.have.property 'cost', 34
        a.carrier.should.have.property '_id'
