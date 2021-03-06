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

       it 'should interpret in the proper timezone', ->
        data =
          '2014-12-05': table: 'pinguy-2014'
          '2016-01-05': table: 'pinguy-2016'
        connect_stamp = moment '2016-01-05 00:30Z'
        a = rating_of data, connect_stamp, 'America/Chicago'
        a.should.deep.equal table:'pinguy-2014'
