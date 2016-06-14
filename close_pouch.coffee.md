    module.exports = close_pouch = (db) ->
      if db.close?
        db.close()
      else
        db.emit 'destroyed'
