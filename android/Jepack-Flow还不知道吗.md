> Fragment 应该始终使用 viewLifecycleOwner 去触发 UI 更新. 但是这不适用于 DialogFragment. 因为它可能没有 View. 对于 DialogFragment. 你应该使用 lifecycleOwner

[Kotlin Flow 一种更安全的 UI 层收集流的方式](https://blog.csdn.net/u011692041/article/details/117552958)

[从LiveData迁移到Kotlin的 Flow,才发现是真的香！](https://blog.csdn.net/A_pyf/article/details/119044687?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_title~default-1.control&spm=1001.2101.3001.4242)

