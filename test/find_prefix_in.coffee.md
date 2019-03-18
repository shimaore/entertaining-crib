    {expect} = chai = require 'chai'
    chai.should()
    CouchDB = require 'most-couchdb'
    prefix = 'http://admin:password@couchdb:5984'

    describe 'find_prefix_in', ->
      find_prefix_in = require '../lib/find_prefix_in'

      it 'should return nothing if none is available', ->
        db = new CouchDB prefix+'/'+'test1'
        await db.create()
        after -> db.destroy()
        a = await find_prefix_in '33972222713', db
        expect(a).to.be.null

      it 'should return nothing if no valid is avalaible', ->
        db = new CouchDB prefix+'/'+'test2'
        await db.create()
        after -> db.destroy()
        await db.put _id: '339'
        await db.put _id: 'prefix:3398'
        a = await find_prefix_in '33972222713', db
        expect(a).to.be.null

      it 'should return one if one is available', ->
        db = new CouchDB prefix+'/'+'test3'
        await db.create()
        after -> db.destroy()
        await db.put _id: 'prefix:339'
        a = await find_prefix_in '33972222713', db
        (expect a).to.have.property '_id', 'prefix:339'

      it 'should return the longest one', ->
        db = new CouchDB prefix+'/'+'test4'
        await db.create()
        after -> db.destroy()
        await db.put _id: 'prefix:339'
        await db.put _id: 'prefix:33972'
        await db.put _id: 'prefix:3397222'
        a = await find_prefix_in '33972222713', db
        (expect a).to.have.property '_id', 'prefix:3397222'
