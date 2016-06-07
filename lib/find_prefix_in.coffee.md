    seem = require 'seem'

    find_prefix_in = seem (destination,database) ->

      ids = [0..destination.length]
        .map (l) -> "prefix:#{destination[0...l]}"
        .reverse()

      {rows} = yield database.allDocs
        keys: ids
        include_docs: true
      (row.doc for row in rows when row.doc?)[0] ? null

    module.exports = find_prefix_in
