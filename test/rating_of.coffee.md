    chai = require 'chai'
    {expect} = chai
    chai.should()
    moment = require 'moment-timezone'

    describe 'rating_of', ->
      rating_of = require '../lib/rating_of'
      it 'should select none when none is present', ->
        data = {}
        connect_stamp = moment '2014-05-13T13:20Z'
        a = rating_of data, connect_stamp
        expect(a).to.be.null

      it 'should select none when none is valid', ->
        data =
          '2014-12-05': table: 'pinguy-2014'
          '2016-01-05': table: 'pinguy-2016'
        connect_stamp = moment '2014-05-13T13:20Z'
        a = rating_of data, connect_stamp
        expect(a).to.be.null

      it 'should select the proper one in an interval', ->
        data =
          '2014-12-05': table: 'pinguy-2014'
          '2016-01-05': table: 'pinguy-2016'
        connect_stamp = moment '2015-05-13T13:20Z'
        a = rating_of data, connect_stamp
        a.should.deep.equal table:'pinguy-2014'

       it 'should select the proper one after an interval', ->
        data =
          '2014-12-05': table: 'pinguy-2014'
          '2016-01-05': table: 'pinguy-2016'
        connect_stamp = moment '2017-05-13T13:20Z'
        a = rating_of data, connect_stamp
        a.should.deep.equal table:'pinguy-2016'

       it 'should select the proper one in a long interval', ->
        data =
          '2008-12-05': table: 'pinguy-2008'
          '2014-12-05': table: 'pinguy-2014'
          '2016-01-05': table: 'pinguy-2016'
          '2018-02-05': table: 'pinguy-2018'
          '2022-03-05': table: 'pinguy-2022'

        connect_stamp = moment '2007-05-13T13:20Z'
        a = rating_of data, connect_stamp
        expect(a).to.be.null

        connect_stamp = moment '2007-05-13T13:20Z'
        a = rating_of data, connect_stamp
        expect(a).to.be.null

        connect_stamp = moment '2009-05-13T13:20Z'
        a = rating_of data, connect_stamp
        a.should.deep.equal table:'pinguy-2008'

        connect_stamp = moment '2017-05-13T13:20Z'
        a = rating_of data, connect_stamp
        a.should.deep.equal table:'pinguy-2016'

        connect_stamp = moment '2018-05-13T13:20Z'
        a = rating_of data, connect_stamp
        a.should.deep.equal table:'pinguy-2018'

        connect_stamp = moment '2021-05-13T13:20Z'
        a = rating_of data, connect_stamp
        a.should.deep.equal table:'pinguy-2018'

        connect_stamp = moment '2024-05-13T13:20Z'
        a = rating_of data, connect_stamp
        a.should.deep.equal table:'pinguy-2022'

       it 'should interpret in the proper timezone', ->
        data =
          '2014-12-05': table: 'pinguy-2014'
          '2016-01-05': table: 'pinguy-2016'
        connect_stamp = moment '2016-01-05 00:30Z'
        a = rating_of data, connect_stamp, 'America/Chicago'
        a.should.deep.equal table:'pinguy-2014'
