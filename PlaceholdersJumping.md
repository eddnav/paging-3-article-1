# Paging 3's Placeholders and Jumping Features

(Royalty-free hipstery photo or art showcasing pages)

With the release of Paging 3, a host of new features have been made available: from first-class support for coroutines and Flow to data stream transformations, to improvements for reactive loading state. Most of these features, alongside migration guides, are prominently featured on the official documentation. However, some others and returning ones are just briefly mentioned or omitted from the CodeLabs for the sake of simplicity. In this article, I'm hoping to expand a bit on two of these: placeholders and the jumping functionality they enable.

Note: This article assumes the reader has basic knowledge of the Paging 3 library. Checking the official documentation (https://developer.android.com/topic/libraries/architecture/paging/v3-overview) is a great way to get started.


## Data set requirements

The most important point to consider whenever attempting to implement placeholders and jumping with Paging 3 is that our data set needs to be countable, this means that we should know exactly how many items the list can have in its entirety. If your data set source is completely network-based, your API responses should provide you with a count property or another mechanism to get this metadata.

```
{
  "total": 3391, // This is what we want
  "items": [
    ...
  ]
}
```

## Placeholders

Placeholders are UI elements that we can show while awaiting the loading of our real pages and items. They make for a nice UI pattern where lists are uninterrupted and user interaction is not limited by loading new elements. Aside from that, because we can show the entire list from the get-go, we are also able to better support scrollbars and, after the first page is loaded, there's no longer the need to show loading spinners on footers. Instead, the placeholders themselves convey this scenario.

(Gif1)

So, how do we enable them? well, much like in Paging 2, they are enabled by default (though they can be disabled via `Pager.enablePlaceholders`), but for the feature to work our `PagingSource` also needs to be able to "count" placeholders. This is done each time we load a new page through `LoadResult.Page` and its properties `itemsBefore` and `itemsAfter`. By analyzing our resolved page, we can count how many items we have before and after, and from this information, Paging can appropriately fill the paging data with placeholders.

```
 override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Int> {
        val pageIndex = checkNotNull(params.key) { "key should not be null" }
        val loadSize = params.loadSize
        val page = pageRepository.getPage(pageIndex, loadSize)

        val newCount = page.items.size
        val total = page.total
        val itemsBefore = pageIndex * PAGE_SIZE
        val itemsAfter = total - (itemsBefore + newCount)

        val prevKey = if (pageIndex == 0) null else pageIndex - 1
        val nextKey = if (itemsAfter == 0) null else pageIndex + 1

        return LoadResult.Page(
            data = page.items,
            prevKey = prevKey,
            nextKey = nextKey,
            itemsBefore = itemsBefore,
            itemsAfter = itemsAfter,
        )
    }
}
```
```

Note: bear in mind this example assumes a naive integer index-based pagination structure, "counting" your items before or after might look very different from this in your implementation.

After setting up our `PagingSource` properly, it's just a matter of handling the placeholders in the UI. The placeholder elements will be represented as `null` in the data and with this knowledge, we can show the appropriate UI.

```
@Composable
fun List(pagingDataFlow: Flow<PagingData<Int>>) {
    val lazyPagingItems = pagingDataFlow.collectAsLazyPagingItems()
    LazyColumn(
        content = {
            items(lazyPagingItems.itemCount) { index ->
                val item = lazyPagingItems[index]
                if (item != null) {
                    ActualItem(item)
                } else {
                    PlaceholderItem()
                }
            }
        }
    )
}
```

 For this example, I used Compose to simplify the UI side of things, however, it's important to note that Paging 3 has complete support for the good old `RecyclerView` through `PagingDataAdapter`. The only thing to keep in mind is that your view holders must be able to support binding nullable values, everything else should be exactly the same whether you choose to use Compose or not.


## Jumping

If your data set is long enough, right away, after setting up your placeholders and giving them a spin you might notice something that might be problematic. What happens if the user scrolls fast into pages that are not yet loaded?

(Gif2)

That's right, the loading of the page the user might be interested in will take a long time because Paging will load each page sequentially until it catches up with the user. This is where the jumping feature in Paging 3 comes in handy. Jumping essentially enables starting to load from an arbitrary page. To support this feature, we first need to support placeholders. From there we start by enabling jumping by overriding `jumpingSupported` in our `PagingSource`.

```
class ExamplePagingSource : PagingSource<Int, Int>() {

    override val jumpingSupported: Boolean = true

    ...
}
```

We also need to properly implement [`getRefreshKey`](https://developer.android.com/reference/kotlin/androidx/paging/PagingSource#getRefreshKey(androidx.paging.PagingState)).


```
class ExamplePagingSource : PagingSource<Int, Int>() {

 override fun getRefreshKey(state: PagingState<Int, Int>): Int? {
        val anchorPosition = state.anchorPosition ?: return null
        val pageIndex = anchorPosition / PAGE_SIZE

        return pageIndex
    }

    ...
}
```

 Whenever a jump happens, Paging will invalidate itself and start loading from scratch based on the key we return from `getRefreshKey`. Similar to how we "count" placeholders, this will depend on your pagination implementation. In this example, our keys are simply an integer that represents the index of our pages; thus, we can get `anchorPosition` (last accessed index at the time of invalidation) and compute the page index closest to it.

 Lastly, we need to set a `jumpingThreshold` in our `PagingConfig`. This property defines how many items past loaded items Paging will consider loading incrementally before giving up and invalidating. Tweaking this value is key to getting a nice experience. If set too low invalidation can happen for way too small jumps, conversely, if set too high invalidation will not happen for big jumps.

 ```
Pager(
    config = PagingConfig(
        ...
        jumpThreshold = PAGE_SIZE * 3,
    ),
    ...
)
```

The result should look like this:

(Gif3)


## Conclusion

Depending on your use case, placeholders and jumping functionality might be very useful, but getting started can be tricky due to their curious omission from the official guides and the fact that many variables affect a given implementation (data source origin, countability mechanism, pagination mechanism). Hopefully, this article cleared up some of its basics for you.

---
##### Maybe a hook to our careers page/recruiting?
e.g. "Want to explore more of Paging and other cool stuff with me and my colleagues? Albert Heijn is hiring (https://werk.ah.nl/vacatures/?filter-searchphrase=Android&filter-experience-level=&address=)!"
