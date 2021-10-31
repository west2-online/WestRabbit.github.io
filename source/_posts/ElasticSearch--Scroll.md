| title                             | date       | author | catalog | tags |
| --------------------------------- | ---------- | ------ | ------- | ---- |
| ElasticSearch的滚动查询--(Scroll) | 2021-10-30 | 02     | true    | Java |



## 为什么要使用Scroll滚动查询?

- `Elasticsearch`是一款优秀的开源企业级搜索引擎，其查询接口主要为Search接口，提供了丰富的各类查询、排序、统计聚合等功能。
- 但是由于它的机制，在做分页的时候要特别注意，我们构造各种条件，可以进行各种查询，但是如果数据量大，不知道大家发现一个问题没，那就是你**`getHits`，只能get到一万**
- 为什么呢？因为之前做的分页前提是都查出来，排序，全都怼进内存里，相当于MySQL里的limit，显然想要高速取出更高量的数据是不合理的
- 举个例子，假如每页放 10 条数据，现在要查询第 100 页，实际上是会把每个` shard` 上存储的前 1000 条数据都查到一个协调节点上，如果你有个 5 个 `shard`，那么就有 5000 条数据。你翻页的时候，翻的越深，每个` shard `返回的数据就越多，而且协调节点处理的时间也就越长。所以用 es 做分页的时候，会发现越翻到后面，就越是慢。
- 那有什么解决方案吗？

### SearchAfter

- `Search`接口另一种翻页方式是`SearchAfter`，时间复杂度O(n)，空间复杂度O(1)。
- `SearchAfter`作为一种动态指针的技术，每次查询都会携带上一次的排序值，这样下次取结果只需要从上次的位点继续扫数据。`SearchAfter`的翻页方式在性能上有了质的提升，但是其限制了用户只能一页一页往后翻，无法跳页，类似于 app 里的推荐商品不断下拉出来新的一页，所以现在很多产品，都是不允许随意翻页的，你能做的就是往下拉。

### SearchScroll

- `Search`接口在使用`SearchAfter`后，相比size+from的翻页方式，翻页性能有质的提升，但是和`SearchScroll`相比，性能逊色很多，用户需要获取的数据越多，翻的越深，则差别越大。
- `SearchScroll`的翻页方式，时间复杂度O(1)，空间复杂度O(1)。
- `SearchScroll`请求天然支持多并发方式查询，因此`SearchScroll`特别适合批量快速拉取大量数据，然后交给spark等计算平台进行后续数据分析处理。它允许我们先做查询初始化，然后再批量地拉取结果。 这有点儿像传统数据库中的 *cursor* 。

## Scroll的原理

- 第一阶段：和传统的Search请求流程几乎一致，在Search流程的基础上进行了一些额外的特殊处理，比如Slice并发处理、Context上下文保留、Response中返回scroll_id、记录本次的游标地址方便下一次scroll请求继续获取数据等等。
- 第二阶段：Scroll请求大大简化，Search中的许多流程都不要再次进行，仅需要执行query、fetch、response三个阶段。而完整的search请求包含rewrite、can_match、dfs、query、fetch、dfs_query、expand、response等复杂的流程。

`其中很重要的一部分，就是在创建SearchContext的时候`

### CreateContext

- 创建`SearchContex`后，如果是`scroll`请求，则在`searchContext`中设置`ScrollContext`。`ScrollContext`中主要包含`context`的有效时间、上一次访问了哪个文档等信息

```java
private Map<String, Object> context = null;
public long totalHits = -1;
public float maxScore;
public ScoreDoc lastEmittedDoc;
public Scroll scroll;
```

## Scroll的简单使用

- 为了使用 `scroll`，初始搜索请求应该在查询中指定 `scroll` 参数，这可以告诉 `Elasticsearch` 需要保持搜索的上下文环境多久，如 `?scroll=1m`。

- ```java
  curl -XGET 'localhost:9200/west2/try/_search?scroll=1m' -d '
  {
      "query": {
          "match" : {
              "title" : "elasticsearch"
          }
      }
  }
  ```

- 使用上面的请求返回的结果中包含一个 `scroll_id`，它是一个base64编码的长字符串 ，这个 ID 可以被传递给 `scroll` API 来检索下一个批次的结果。

- ```java
  curl -XGET  'localhost:9200/_search/scroll'  -d'
  {
      "scroll" : "1m", 
      "scroll_id" : "cXVlcnlUaGVuRmV0Y2g7NTsxMDk5NDpkUmpiR2FjOFNhNnlCM1ZDMWpWYnRROzEwOTk1OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MTA5OTM6ZFJqYkdhYzhTYTZ5QjNWQzFqVmJ0UTsxMTE5MDpBVUtwN2lxc1FLZV8yRGVjWlI2QUVBOzEwOTk2OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MDs=" 
  }
  ```

- 每次对 `scroll` API 的调用返回了结果的下一个批次知道没有更多的结果返回，也就是直到 `hits` 数组空了。

- 注意查询每次返回一个新字段 `_scroll_id`。每次我们做下一次查询， 我们必须把前一次查询返回的字段 `_scroll_id` 传递进去。

## 结语

- 详细的整合使用就不在此赘述，可以尝试创建`elasticsearchTemplate`的bean，使用两个核心方法`elasticsearchTemplate.startScroll()` 与 `elasticsearchTemplate.continueScroll()`

- Search和Scroll的核心区别，主要是在query阶段需要处理并发的scroll请求，fetch阶段需要得到本次返回给用户的最后一个文档，然后告知data节点的context，这样下次请求就可以继续从上一个记录点进行搜索。

  

