Documentation
------

redux-search provides the following named exports:

* [`createSearchAction`](#createsearchactionresourcename)
* [`getSearchSelectors`](#getsearchselectors-filterfunction-resourcename-resourceselector-searchstateselector-)
* [`reducer`](#reducer)
* [`reduxSearch`](#reduxsearch-resourceindexes-resourceselector-searchapi-searchstateselector-)
* [`CapabilitiesBasedSearchApi`](#searchapi--workersearchapicapabilitiesbasedsearchapi)
* [`SearchApi`](#searchapi--workersearchapi)
* [`SearchUtility`](#searchutility)
* [`WorkerSearchApi`](#searchapi--workersearchapi)

### `createSearchAction(resourceName)`
Factory function that creates Redux search actions. This function requires a single parameter (a `resourceName` string) that identifies a searchable resource. For example:

```javascript
import { createSearchAction } from 'redux-search'

const actions = {
  searchBooks: createSearchAction('books')
}
```

### `getSearchSelectors({ filterFunction, resourceName, resourceSelector, searchStateSelector })`

Creates selectors for a searchable resource. This function returns an object with selectors defined as the following properties:

* **text**: Selector that returns the current search text for a given searchable resource.
* **result**: Selector that returns the current result list for a given searchable resource. This list is pre-filtered to ensure that all ids exist within the current resource collection.
* **unfilteredResult**: Selector that returns the current result list for a given searchable resource. This list is not pre-filtered and should only be used for computed resources.

This method accepts the following named parameters:

##### filterFunction

Custom filter function for computed resources. A default filter function capable of supporting plain Objects and immutable Maps is provided. If you provide your own filter function it should have the following signature: `(id: string): boolean`

##### resourceName
Identifies a searchable resource (eg. 'books')

##### resourceSelector:
Selector responsible for returning an iterable resource map for a given, searchable resource. This function should be capable of returning a map for each resource listed in `resourceIndexes`. Its signature should look like this: `(resourceName: string, state: Object): Iterable<Object>`

##### searchStateSelector
Responsible for returning the Search state. A default implementation is provided. Override only if you add `searchReducer()` to the store somewhere other than `state.search`.

For example:

```javascript
import { connect } from 'react-redux'
import { createSelector } from 'reselect'
import { getSearchSelectors } from 'redux-search'

const { text, result } = getSearchSelectors('books')

const selectors = createSelector(
  [result, books, text],
  (result, books, text) => ({
    bookIds: result,
    books,
    searchText
  })
)
```

### `reducer`

The root reducer for redux-search. It is recommended that you add this to the application state as `search`. For example:

```javascript
import { combineReducers } from 'redux'
import { reducer } from '../src/index'

const rootReducer = combineReducers({
  search: reducer
  // Your other reducers go here...
})
```

If you add the reducer at an alternate location you will need to supply a `searchStateSelector` to `reduxSearch()` and `getSearchSelectors()`.

### `reduxSearch({ resourceIndexes, resourceSelector, searchApi, searchStateSelector })`
Creates higher-order search store to be composed with other store enhancers.
This function accepts the following named parameters (via a params Object):

##### resourceIndexes:
Maps searchable resources to search configurations. Configurations must be one of the following types:

1. an array of searchable field names (eg. `["name", "description"]`)
2. a custom indexing function ([read more about that here](reduxSearch.md))

##### resourceSelector:
Selector responsible for returning an iterable resource map for a given, searchable resource. This function should be capable of returning a map for each resource listed in `resourceIndexes`. Its signature should look like this: `(resourceName: string, state: Object): Iterable<Object>`

If this value is specified then the search index will be automatically built/updated whenever resources change. Ommit this property if you wish to manage the search index manually.

##### Search:
Observable Search API. Should only be overridden for testing purposes. Refer to [`SearchApi`](#searchapi--workersearchapi) for more information if you choose to override this.

##### searchStateSelector:
Selects the search sub-state within the state store. A default implementation is provided. Override only if you add `searchReducer()` to the store somewhere other than `state.search`.

### `SearchUtility`

Forked from [JS search](github.com/bvaughn/js-search), this utility builds a search index and runs actual searches. One instance of `SearchUtility` is created for each searchable resource type in redux-search. Depending on how [`reduxSearch`](#reduxsearch-resourceindexes-resourceselector-searchapi-searchstateselector-) has been configured this instance may live in the UI thread or in a web-worker.

In most cases this utility does not need to be used directly. redux-search will automatically create and configure searchable instances in response to store changes. The utility is exposed though in case custom search functionality is needed that falls outside of the primary scope of redux-search. In that case you can build and run your own searches using the following methods:

##### `indexDocument (uid, text)`
Adds or updates a uid in the search index and associates it with the specified text. Note that at this time uids can only be added or updated in the index, not removed.

* **uid**: Uniquely identifies a searchable object
* **text**: Searchable text to associate with the uid

##### `search(query)`
Searches the current index for the specified query text. Only uids matching all of the words within the text will be accepted. If an empty query string is provided all indexed uids will be returned.

Document searches are case-insensitive (e.g. "search" will match "Search"). Document searches use substring matching (e.g. "na" and "me" will both match "name").

* **query**: Searchable query text

This method will return an array of uids.

### `SearchApi` / `WorkerSearchApi` / `CapabilitiesBasedSearchApi`
+The search API is an observable that manages communication between the redux-search middleware and the underlying search utility. It maps resource names to search indicies and coordinates searches. Both a single-threaded implementation (`SearchApi`) and a web-worker implementation (`WorkerSearchApi`) are provided.

By default a capabilities-based search API is used (`CapabilitiesBasedSearchApi`) that auto-detects web worker support and uses the best implementation for the current environment. You can override this behavior with `reduxSearch()` for testing purposes like so:

```javascript
import { SearchApi } from './SearchApi'

reduxSearch({
  // Configure :resourceIndexes and :resourceSelector here and then...
  searchApi = new SearchApi()
})
```

You may also want to override this value for unit testing purposes. To do so you will need to implement the following methods:

##### `subscribe (onNext, onError)`
Subscribe to Search events. Subscribers will be notified each time a Search is performed.

Successful searches will call `onNext` with the following parameters:
* **result**: An array of ids matching the search
* **text**: Search string
* **resourceName**: Identifies the resource that was searched

Failed searches (searches that result in an Error) will call `onError` with an `Error` parameter.

This method returns a callback that can be used to unsubscribe from Search events. Just invoke the function without any parameters to unsubscribe.

##### `indexResource (resourceName, fieldNamesOrIndexFunction, resources)`
Builds a searchable index of a set of resources. It accepts the following parameters:

* **resourceName** Uniquely identifies the resource (eg. "databases")
* **fieldNamesOrIndexFunction** This value is passed to reduxSearch() factory during initialization. It is either an Array of searchable fields (to be auto-indexed) or a custom index function to be called with a `resources` object and an `indexDocument` callback function.
* **resources** Map of resource id to resource (Object)

##### `async performSearch (resourceName, text)`
Searches a resource and returns a Promise to be resolved with an array of ids that match the search string. Upon completion (or failure) this method also notifies all current subscribers. It accepts the following parameters:

* **resourceName** Uniquely identifies the resource (eg. "books")
* **text** Search string
