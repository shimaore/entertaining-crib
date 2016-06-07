    seem = require 'seem'
    {expect} = chai = require 'chai'
    chai.should()
    PouchDB = (require 'pouchdb').defaults db: require 'memdown'

    describe 'Rating', ->
      Rating = require '..'
      prov = new PouchDB 'prov'

      it 'should not rate unless client or carrier are specified', seem ->
        r = new Rating {prov}, 'test1'
        a = yield r.rate
          direction: 'egress'
          from: '33972222713'
          to: '18002255288'
          stamp: '2013-05-14 12:52:23'
          duration: 23
          source_id: 'd895b2ea-58fe-4215-aae7-ca66a6c3099c'
        a.should.have.length 0

      it 'should rate carrier', seem ->
        r = new Rating {prov}, 'test2', PouchDB
        yield prov.put
          _id: 'carrier:default'
          timezone: 'America/New_York'
          rating:
            '2012-12-14': table:'cheap'
        cheap = new PouchDB 'tarif-cheap'
        yield cheap.put
          _id:'configuration'
          currency: 'EUR'
          divider: 1000
          per: 60
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
          carrier: 'default'
        a.should.have.length 1
        a[0].should.have.property 'billable_number', '33972222713'
        a[0].should.have.property 'remote_number', '18002255288'
        a[0].should.have.property 'source', 'test2'
        a[0].should.have.property 'currency', 'EUR'
        a[0].should.have.property 'periods', 23
        a[0].should.have.property 'integer_amount', 14
        a[0].should.have.property 'divider', 1000
        a[0].should.have.property 'subsequent'
        a[0].subsequent.should.have.property 'cost', 34
        a[0].should.have.property '_id'
        a[0].should.have.property '_target_db'

      it 'should rate client', seem ->
        r = new Rating {prov}, 'test3', PouchDB
        yield prov.put
          _id: 'account:client-1'
          timezone: 'America/New_York'
          rating:
            '2012-12-14': table:'expensive'
        expensive = new PouchDB 'tarif-expensive'
        yield expensive.put
          _id:'configuration'
          currency: 'EUR'
          divider: 100
          per: 60
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
          client: 'client-1'
        a.should.have.length 1
        a[0].should.have.property 'billable_number', '33972222713'
        a[0].should.have.property 'remote_number', '19005551212'
        a[0].should.have.property 'source', 'test3'
        a[0].should.have.property 'currency', 'EUR'
        a[0].should.have.property 'periods', 1
        a[0].should.have.property 'integer_amount', 750
        a[0].should.have.property 'divider', 100
        a[0].should.have.property 'subsequent'
        a[0].subsequent.should.have.property 'cost', 1500
        a[0].should.have.property '_id'
        a[0].should.have.property '_target_db'

      it 'should rate client and carrier', seem ->
        r = new Rating {prov}, 'test4', PouchDB
        yield prov.put
          _id: 'account:2'
          timezone: 'America/New_York'
          rating:
            '2012-12-14': table:'client-2'
        yield prov.put
          _id: 'carrier:A'
          timezone: 'America/New_York'
          rating:
            '2012-12-14': table:'carrier-A'
        expensive = new PouchDB 'tarif-client-2'
        yield expensive.put
          _id:'configuration'
          currency: 'EUR'
          divider: 100
          per: 60
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
        cheap = new PouchDB 'tarif-carrier-A'
        yield cheap.put
          _id:'configuration'
          currency: 'EUR'
          divider: 1000
          per: 60
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
          client: '2'
          carrier: 'A'
        a.should.have.length 2
        a[0].should.have.property 'billable_number', '33972222713'
        a[0].should.have.property 'remote_number', '19005551212'
        a[0].should.have.property 'source', 'test4'
        a[0].should.have.property 'currency', 'EUR'
        a[0].should.have.property 'periods', 1
        a[0].should.have.property 'integer_amount', 750
        a[0].should.have.property 'divider', 100
        a[0].should.have.property 'subsequent'
        a[0].subsequent.should.have.property 'cost', 1500
        a[0].should.have.property '_id'
        a[0].should.have.property '_target_db'
        a[1].should.have.property 'billable_number', '33972222713'
        a[1].should.have.property 'remote_number', '19005551212'
        a[1].should.have.property 'source', 'test4'
        a[1].should.have.property 'currency', 'EUR'
        a[1].should.have.property 'periods', 23
        a[1].should.have.property 'integer_amount', 131
        a[1].should.have.property 'divider', 1000
        a[1].should.have.property 'subsequent'
        a[1].subsequent.should.have.property 'cost', 340
        a[1].should.have.property '_id', '33972222713-2013-05-14T12:52:23-04:00-19005551212-23'
        a[1].should.have.property '_target_db', 'rated-A-2013-05'
