## RecyclerView 的缓存机制


`Recycler`有4个层次用于缓存`ViewHolder`对象，优先级从高到底依次为`ArrayList<ViewHolder> mAttachedScrap`、`ArrayList<ViewHolder> mCachedViews`、`ViewCacheExtension mViewCacheExtension`、`RecycledViewPool mRecyclerPool`。如果四层缓存都未命中，则重新创建并绑定`ViewHolder`对象

**缓存性能：**

| 缓存           | 重新创建`ViewHolder` | 重新绑定数据 |
| -------------- | -------------------- | ------------ |
| mAttachedScrap | false                | false        |
| mCachedViews   | false                | false        |
| mRecyclerPool  | false                | true         |

**缓存容量：**

- `mAttachedScrap`：没有大小限制，但最多包含屏幕可见表项。
- `mCachedViews`：默认大小限制为2，放不下时，按照先进先出原则将最先进入的`ViewHolder`存入回收池以腾出空间。
- `mRecyclerPool`：对`ViewHolder`按`viewType`分类存储（通过`SparseArray`），同类`ViewHolder`存储在默认大小为5的`ArrayList`中。


**缓存用途：**

- `mAttachedScrap`：用于布局过程中屏幕可见表项的回收和复用。
- `mCachedViews`：用于移出屏幕表项的回收和复用，且只能用于指定位置的表项，有点像“回收池预备队列”，即总是先回收到`mCachedViews`，当它放不下的时候，按照先进先出原则将最先进入的`ViewHolder`存入回收池。
- `mRecyclerPool`：用于移出屏幕表项的回收和复用，且只能用于指定`viewType`的表项

1. **缓存结构：**

   - `mAttachedScrap`：`ArrayList<ViewHolder>`
   - `mCachedViews`：`ArrayList<ViewHolder>`
   - `mRecyclerPool`：对`ViewHolder`按`viewType`分类存储在`SparseArray<ScrapData>`中，同类`ViewHolder`存储在`ScrapData`中的`ArrayList`中

   

https://juejin.cn/post/6844903780006264845

