    {expect} = chai = require 'chai'
    chai.should()
    seem = require 'seem'
    PouchDB = require 'pouchdb-core'
      .plugin require 'pouchdb-adapter-memory'
      .defaults adapter:'memory'

    describe 'find_prefix_in', ->
      find_prefix_in = require '../lib/find_prefix_in'

      it 'should return nothing if none is available', seem ->
        db = new PouchDB 'test1'
        a = yield find_prefix_in '33972222713', db
        expect(a).to.be.null

      it 'should return nothing if no valid is avalaible', seem ->
        db = new PouchDB 'test2'
        yield db.put _id: '339'
        yield db.put _id: 'prefix:3398'
        a = yield find_prefix_in '33972222713', db
        expect(a).to.be.null

      it 'should return one if one is available', seem ->
        db = new PouchDB 'test3'
        yield db.put _id: 'prefix:339'
        a = yield find_prefix_in '33972222713', db
        a.should.have.property '_id', 'prefix:339'

      it 'should return the longest one', seem ->
        db = new PouchDB 'test4'
        yield db.put _id: 'prefix:339'
        yield db.put _id: 'prefix:33972'
        yield db.put _id: 'prefix:3397222'
        a = yield find_prefix_in '33972222713', db
        a.should.have.property '_id', 'prefix:3397222'
