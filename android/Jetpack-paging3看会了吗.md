听到Paging3的时候，觉得这是个分页加载库，自己早就封装了一套，可学可不学吧。后来看到很多项目有用到这个，就想着它到底哪里方便了。

平时在加载数据的时候需要手动实现这些功能：

1. 跟踪用户滑到末尾请求
2. 确保多个请求不会同事触发
3. 对数据进行缓存
4. 跟踪加载状态，在列表上显示加载状态，如果有失败的加载，可重试的加载

而在paging3中则不需要考虑滑动底部的时候发起一个网络请求加载下一页数据，它是一个全新的思路，在学习paging3的时候关联了`协程`、`Flow`、`DiffUtil`、`MVVM`等知识点。

首先我们先了解下paging3中几个主要组件:

**`pagingData`**：用于存储分页数据的容器

**`PagingSource`**：用于将数据加载到`PagingData` 流的基类

**`Pager.flow`**:：根据 `PagingConfig` 和一个定义如何构造实现的 `PagingSource` 的构造函数，构建一个 `Flow<PagingData>`

**`PagingDataAdapter`**：用于在 `RecyclerView` 中呈现 `PagingData` 的 `RecyclerView.Adapter`,必须实现

**`RemoteMediator`** ： 帮助接收来自网络和数据库的数据，实现分页 (`后续进行分析`)

**使用Paging组件可以轻松的在应用的界面中逐步、流畅的的加载数据。**

在撸起袖子干时，先在`build.gradle`中添加依赖库：

```groovy
dependencies {
    def paging_version = "3.0.1"
		...
    //paging3
    implementation "androidx.paging:paging-runtime-ktx:$paging_version"
}
```

> 这次使用的`API`来自`wanandroid`的接口数据，把其中主要的关键类写了下，详细步骤浏览源码

好了，可以愉快干活了~

## 定义数据源

```kotlin
private const val START_INDEX = 0
class CustomPageDataSource(private val wanService: WanService): PagingSource<Int, Article>() {  //第一个参数有 页数的数据源类型， 第二个参数 每一项数据对应的对象类型

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {

        return try {
            val page = params.key ?: START_INDEX   //params.key  当前页数
            val pageSize = params.loadSize    //一页多少数据
            val repoResponse = wanService.getHomeArticles(page)
            val articleList = repoResponse.data!!.datas
            val prevKey = if (page  == START_INDEX) null else page - 1 //如果可以往上加载更多就设置该参数，否则不设置
            val nextKey = if(page == repoResponse.data!!.pageCount) null else page + 1  ////加载下一页的key 如果传null就说明到底了
            LoadResult.Page(articleList , prevKey, nextKey)  //构建一个LoadResult对象并返回

        } catch (e: Exception){
            e.printStackTrace()
            LoadResult.Error(e)
        }
    }

    @ExperimentalPagingApi
    override fun getRefreshKey(state: PagingState<Int, Article>): Int? = null  //高级的用法

}
```

可以看到`PaginSource`实现两个函数`load`和`getRefreshKey`，在滚动的过程中会以异步的方式调用`load`方法获取数据源。

`Loadparams`对象保存了`要加载页面的页数`，其中第一次调用的时候将会是null，所以我们要对它进行初始值赋值`params.key ?: START_INDEX`。获取加载的数据大小则是`params.loadSize`。

最后调用了`LoadResult.Page`构建一个`LoadResult`返回。第二个参数和第三个参数分贝对应上一页和下一页的页数。

**注意点**：在第一页和最后一页的时候传null。

## 构建和缓存PagingData

```kotlin
object Repository {
    private const val PAGE_SIZE = 20
    fun getArticleList(): Flow<PagingData<Article>> {
        return Pager(
            config = PagingConfig(pageSize = PAGE_SIZE),   //每页所包含的数据量
            pagingSourceFactory = {CustomPageDataSource(WanRetrofitClient.service)}  //作为用于分页的数据源
        ).flow  //转成Flow类型
    }
}
```

`PagingConfig`这个类定义了加载内容的选项，如多久加载，加载请求的数量。默认情况下，如果 Paging 可以统计未加载项的数量以及`enablePlaceholders` 配置标志为 true，那么 Paging 将返回 null 作为尚未加载内容的占位符。

## 在ViewModel中请求

```kotlin
class Paging3ViewModel: BaseViewModel() {
    fun getPagingData(): Flow<PagingData<Article>> {
        //cachedIn  用于将服务器返回的数据在viewModelScope这个作用域内进行缓存
        return Repository.getArticleList().cachedIn(viewModelScope)
    }
}
```

这里要说下额外代用了`cachedIn(viewModelScope)`,用于将数据在viewModelScope这个作用域内进行缓存。这样当页面发生旋转时重新创建时，就会直接读取缓存。

## PagingDataAdapter的实现

最关键的来了，在paging3中的适配器必须要事先`PagingDataAdapter`。

```kotlin
class PagingArticleAdapter: PagingDataAdapter<Article, PagingArticleAdapter.ViewHolder>(COMPARATOR) {

    companion object{
        private val COMPARATOR = object : DiffUtil.ItemCallback<Article>(){
            override fun areItemsTheSame(oldItem: Article, newItem: Article): Boolean {
                return oldItem.id == newItem.id
            }
            override fun areContentsTheSame(oldItem: Article, newItem: Article): Boolean {
                return oldItem == newItem
            }
        }
    }

    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val name: TextView = itemView.findViewById(R.id.tv_article_author)
        val title: TextView = itemView.findViewById(R.id.tv_article_title)
        val chapterName: TextView = itemView.findViewById(R.id.tv_article_chapterName)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val repo = getItem(position)
        if (repo != null) {
            holder.name.text = repo.shareUser
            holder.title.text = repo.title
            holder.chapterName.text = repo.superChapterName + repo.chapterName
        }

    }

    override fun onCreateViewHolder(
        parent: ViewGroup,
        viewType: Int
    ): ViewHolder {
        val view = LayoutInflater.from(parent.context).inflate(R.layout.item_paging_homeliist, parent, false)
        return ViewHolder(view)
    }
}
```

在`Paging 3`在内部会使用DiffUtil来管理数据变化， 所以要提供`COMPARATOR`这么个东西,其他的跟平时创建`RecycleView.Adaper`没什么区别。再仔细看看是不是少了什么没有实现，对不需要重写`getItemCount`函数了，`paging3`页在内部帮我们管理了。

## 网络调用数据更新

```kotlin
class PagingDemoActivity: BaseModelActivity<Paging3ViewModel>() {
    
    private val pagingArticleAdapter = PagingArticleAdapter()
    override fun useLoadSir(): Boolean = false
    override fun providerVMClass(): Class<Paging3ViewModel> = Paging3ViewModel::class.java
    override fun getLayoutResId(): Int = R.layout.activity_paging3_main

    override fun initView() {
        recycleview?.run {
            adapter = pagingArticleAdapter.withLoadStateFooter(FooterAdapter{pagingArticleAdapter.retry()})
        }

        pagingArticleAdapter.addLoadStateListener {
            when(it.refresh){
                is LoadState.NotLoading ->{
                    showPageContent()
                    progress_bar.visibility = View.INVISIBLE
                    recycleview.visibility = View.VISIBLE

                }
                is LoadState.Loading -> {
                    progress_bar.visibility = View.VISIBLE
                    recycleview.visibility = View.INVISIBLE
                }
                is LoadState.Error -> {
                    val state = it.refresh as LoadState.Error
                    progress_bar.visibility = View.INVISIBLE
                    toast("Load Error: ${state.error.message}")
                }

            }
        }
    }

    private fun refresh() {
        lifecycleScope.launchWhenCreated {
            mViewModel.getPagingData().collectLatest {
                pagingArticleAdapter.submitData(pagingData = it)  //触发Paging 3分页功能的核心  协程挂起（suspend）操作
            }
        }
    }

    override fun initData() {
       refresh()
    }
}
```

其中里面的`submit()`是触发Paging3分页功能的核心，当状态发生变化时，Flow会通过`CombinedLoadStates` 对象向我们发送相应信息。

可以看到我们设置了个监听回调，`CombinedLoadStates`有三种不同类型的加载状态：

- `CombinedLoadStates.refresh`: 表示首次加载 `PagingData` 的加载状态。
- `CombinedLoadStates.prepend`:表示在列表开头加载数据时的加载状态。
- `CombinedLoadStates.append`:表示在列表末尾加载数据的加载状态。

**刷新回到初始位置的特别的方法**：

```kotlin
private fun initSearch(query: String) {
    ...
        lifecycleScope.launch {
            adapter.loadStateFlow
                   // Only emit when REFRESH LoadState changes.
                   .distinctUntilChangedBy { it.refresh }
                   // Only react to cases where REFRESH completes i.e., NotLoading.
                   .filter { it.refresh is LoadState.NotLoading }
                   .collect { binding.list.scrollToPosition(0) }
        }
}
```

## 底部显示加载状态

添加底部加载做的就是要集成`LoadStateAdapter`,`LoadStateAdapter` 简化了实现，因为它可以在以上两个函数中传递 `LoadState。

```kotlin
class FooterAdapter(val retry:() -> Unit): LoadStateAdapter<FooterAdapter.ViewHolder>() {

    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val progressBar: ProgressBar = itemView.findViewById(R.id.progress_bar)
        val retryButton: Button = itemView.findViewById(R.id.retry_button)
    }

    override fun onBindViewHolder(holder: ViewHolder, loadState: LoadState) {
        holder.progressBar.isVisible = loadState is LoadState.Loading    //正在加载中
        holder.retryButton.isVisible = loadState is LoadState.Error     //加载失败
    }

    override fun onCreateViewHolder(parent: ViewGroup, loadState: LoadState): ViewHolder {
        val view = LayoutInflater.from(parent.context).inflate(R.layout.footer_item, parent, false)
        val holder = ViewHolder(view)
        holder.retryButton.setOnClickListener {
            retry()
        }
        return holder
    }
}
```

这样子我们就拥有了个无限滚动的列表了~

**成果图**



详细的代码可以看这里，**[Paging3Demo](https://github.com/LebronXia/WanAndroid/blob/master/app/src/main/java/com/xiamu/wanandroid/mvvm/demo/paging3/PagingDemoActivity.kt)**

详细的代码可以看这里，**[Paging3Demo](https://github.com/LebronXia/WanAndroid/blob/master/app/src/main/java/com/xiamu/wanandroid/mvvm/demo/paging3/PagingDemoActivity.kt)**

详细的代码可以看这里，**[Paging3Demo](https://github.com/LebronXia/WanAndroid/blob/master/app/src/main/java/com/xiamu/wanandroid/mvvm/demo/paging3/PagingDemoActivity.kt)**

## 花絮

在开发过程中是不是觉得要继承`PagingArticleAdapter`特别不舒服，明明自己有个专属的`BaseAdapter`还不让用。解决也是有办法的。

用个装饰模式，将我们的自己的类传进去就不用特意的再新建个类了，具体使用如下：

```kotlin
/**
 *
 * 自定义PagerAdapter
 */
class PagingWrapAdapter<T: DifferData, VH: RecyclerView.ViewHolder>(
        private val innerAdapter: RecyclerView.Adapter<VH>,
        private val callback: ((List<T>) -> Unit)
) : RecyclerView.Adapter<VH>() {

    private val differ = AsyncPagingDataDiffer<T>(
            diffCallback = com.xiamu.wanandroid.mvvm.demo.paging3.DifferCallback(),
            updateCallback = AdapterListUpdateCallback(this),
            mainDispatcher = Dispatchers.Main,
            workerDispatcher = Dispatchers.Default
    )

    init {
        differ.addLoadStateListener {
            if (it.append is LoadState.NotLoading) {
                val items = differ.snapshot().items
                callback.invoke(items)
            }
        }
    }

    fun addLoadStateListener(listener: (CombinedLoadStates) -> Unit) {
        differ.addLoadStateListener(listener)
    }

    fun withLoadStateFooter(
            footer: LoadStateAdapter<*>
    ): ConcatAdapter {
        addLoadStateListener { loadStates ->
            footer.loadState = loadStates.append
        }
        return ConcatAdapter(this, footer)
    }

    fun retry() {
        differ.retry()
    }

    suspend fun submitList(pagingData: PagingData<T>) {
        differ.submitData(pagingData)
    }


    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        return innerAdapter.onCreateViewHolder(parent, viewType)
    }

    override fun onBindViewHolder(holder: VH, position: Int) {
        differ.getItem(position)
        innerAdapter.onBindViewHolder(holder, position)
    }

    override fun getItemCount(): Int {
        return innerAdapter.itemCount
    }

}
```

可以把原先的`PagingDemoActivity`改写成这样


```kotlin
class PagingDemoActivity: BaseModelActivity<Paging3ViewModel>() {

    private val mAdapter = PagingNormalAdapter()

    private val newPagingArticleAdapter = PagingWrapAdapter<Article, BaseViewHolder>(mAdapter){
        mAdapter.setNewData(it)
    }

   ...
    private fun refresh() {
        lifecycleScope.launchWhenCreated {
            mViewModel.getPagingData().collectLatest {
                newPagingArticleAdapter.submitList(pagingData = it)  //触发Paging 3分页功能的核心  协程挂起（suspend）操作
            }
        }
    }

    override fun initData() {
       refresh()
    }
}
```

## 参考
[Android Paging3官网](https://developer.android.com/codelabs/android-paging#0)



