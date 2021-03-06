## Map/Reduce Example

This is an example of how to use the mapReduce function to perform map/reduce style aggregation on your data.

### Setup

To start, we'll insert some example data which we can perform map/reduce queries on:

    $ ghci
    ...
    Prelude> :set prompt "> "
    > :set -XOverloadedStrings
    > import Database.MongoDB
    > pipe <- connect $ host "127.0.0.1"
    > let run act = access pipe master "test" act
    > let docs = [ ["x" =: 1, "tags" =: ["dog", "cat"]], ["x" =: 2, "tags" =: ["cat"]], ["x" =: 3, "tags" =: ["mouse", "cat", "dog"]] ]
    > run $ insertMany "mr1" docs

### Basic Map/Reduce

Now we'll define our map and reduce functions to count the number of occurrences for each tag in the tags array, across the entire collection.

Our map function just emits a single (key, 1) pair for each tag in the array:

    > let mapFn = Javascript [] "function() {this.tags.forEach (function(z) {emit(z, 1);});}"

The reduce function sums over all of the emitted values for a given key:

    > let reduceFn = Javascript [] "function (key, values) {var total = 0; for (var i = 0; i < values.length; i++) {total += values[i];} return total;}"

Note: We can't just return values.length as the reduce function might be called iteratively on the results of other reduce steps.

Finally, we run mapReduce, results by default will be return in an array in the result document (inlined):

    > run $ runMR' (mapReduce "mr1" mapFn reduceFn)
    [ results: [[ _id: "cat", value: 3.0],[ _id: "dog", value: 2.0],[ _id: "mouse", value: 1.0]], timeMillis: 379, counts: [ input: 4, emit: 6, reduce: 2, output: 3], ok: 1.0]

Inlining only works if result set < 16MB. An alternative to inlining is outputing to a collection. But what to do if there is data already in the collection from a previous run of the same MapReduce? You have three alternatives in the MRMerge data type: Replace, Merge, and Reduce. See its documentation for details. To output to a collection, set the `rOut` field in `MapReduce`.

	> run $ runMR' (mapReduce "mr1" mapFn reduceFn) {rOut = Output Replace "mr1out" Nothing}
	[ result: "mr1out", timeMillis: 379, counts: [ input: 3, emit: 6, reduce: 2, output: 3], ok: 1.0]

You can now query the mr1out collection to see the result, or run another MapReduce on it! A shortcut for running the map-reduce then querying the result collection right away is `runMR`.

	> run $ rest =<< runMR (mapReduce "mr1" mapFn reduceFn) {rOut = Output Replace "mr1out" Nothing}
	[[ _id: "cat", value: 3.0],[ _id: "dog", value: 2.0],[ _id: "mouse", value: 1.0]]
