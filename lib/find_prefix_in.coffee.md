    find_prefix_in = (destination,database) ->

      ids = [0..destination.length]
        .map (l) -> "prefix:#{destination[0...l]}"
        .reverse()

      result = null
      await database
        .query null, '_all_docs',
          keys: ids
          include_docs: true
        .filter ({doc}) -> doc?
        .take 1
        .observe ({doc}) ->
          result = doc
      result

    module.exports = find_prefix_in
