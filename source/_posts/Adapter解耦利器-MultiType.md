---
title: Adapter解耦利器-MultiType
date: 2017-05-09 16:08:07
tags: Adapter
---
# 开始
有段时间没写博客。因为最近公司项目比较忙，所以没什么时间写。最近在做公司的新项目的时候，尝试了一些新的第三方库，后来返现这个MultiType是真的好用，可以节约很多代码，并且还能很好的解耦，具体是怎么使用的，下面我们来慢慢分析~。

<!-- more -->

# 基本用法
这个库的地址是：https://github.com/drakeet/MultiType
使用Gradle就能依赖，这个库的作者已经把使用方式介绍的很清楚看了，详情可以看该库的Wiki。在这里就只说下最基本的用法。

``` java

//三种不同的类型，用来代表不同类的 Item
public class OneType {
}

public class TwoType {
}

public class ThreeType {
}

//编写 ItemViewBinder 类，这里就只写 OneTypeItemViewBinder 其他两种类型的编写方式大同小异。

public class OneTypeItemViewBinder extends ItemViewBinder<OneType,OneTypeItemViewBinder.OneTypeViewHolder>{

    @NonNull
    @Override
    protected OneTypeViewHolder onCreateViewHolder(@NonNull LayoutInflater inflater, @NonNull ViewGroup parent) {
        return new OneTypeViewHolder(inflater.inflate(R.layout.item_one_type,parent,false));
    }

    @Override
    protected void onBindViewHolder(@NonNull OneTypeViewHolder holder, @NonNull OneType item) {

    }

    static class OneTypeViewHolder extends RecyclerView.ViewHolder{
        public OneTypeViewHolder(View itemView) {
            super(itemView);
        }
    }

}

//最后在 Activity 中使用

public class MainActivity extends AppCompatActivity {

    RecyclerView rv_main;

    MultiTypeAdapter mAdapter;

    Items mItems;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //初始化RecyclerView
        rv_main = (RecyclerView) findViewById(R.id.rv_main);
        rv_main.setLayoutManager(new LinearLayoutManager(this));
        rv_main.addItemDecoration(new DividerItemDecoration(this,DividerItemDecoration.VERTICAL));

        //初始化 MultiTypeAdapter 和 Items
        mAdapter = new MultiTypeAdapter();
        mItems = new Items();

        //在 Adapter 中注册 ItemViewBInder
        mAdapter.register(OneType.class,new OneTypeItemViewBinder());
        mAdapter.register(TwoType.class,new TwoTypeItemViewBinder());
        mAdapter.register(ThreeType.class,new ThreeTypeItemViewBinder());

        //setItems
        mAdapter.setItems(mItems);
        rv_main.setAdapter(mAdapter);

        //添加数据
        for (int i = 0; i < 100; i++) {
            mItems.add(new OneType());
            mItems.add(new TwoType());
            mItems.add(new ThreeType());
        }
        //刷新 RecyclerView
        mAdapter.notifyDataSetChanged();

    }
}

```

> 基本用法差不多就是上面那些，这个时候肯定会有人觉得写个Adapter还这么麻烦，还要定义什么ItemViewBinder。其实这里只是一个Demo，虽然代码看上去多了不少，但是可扩展性，复用性大大的提高了，下面我们在用传统的方式实现一个多类型的Adapter，对比对比就知道了。

# 传统实现 VS MultiType

下面我们用传统的方式(可能还有别的方法实现，这里只是我自己实现多类型的 Adapter 的方式)实现一个多类型的Adapter。

``` java

public class SampleRecyclerViewAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    private static final int ITEM_TYPE_ONE_TYPE = 1;

    private static final int ITEM_TYPE_TWO_TYPE = 2;

    private static final int ITEM_TYPE_THREE_TYPE = 3;

    @Override
    public int getItemViewType(int position) {

        if(position>0 && position % 10 == 0){
            return ITEM_TYPE_THREE_TYPE;
        }

        if(position % 2 == 0){
            //偶数
            return ITEM_TYPE_ONE_TYPE;
        }else{
            //基数
            return ITEM_TYPE_TWO_TYPE;
        }
    }

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        if (viewType == ITEM_TYPE_ONE_TYPE) {
            return new OneTypeViewHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout.item_one_type, parent, false));
        } else if (viewType == ITEM_TYPE_TWO_TYPE) {
            return new TwoTypeViewHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout.item_two_type, parent, false));
        } else if (viewType == ITEM_TYPE_THREE_TYPE) {
            return new ThreeTypeViewHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout.item_three_type, parent, false));
        }
        return null;
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
        if (holder instanceof OneTypeViewHolder) {
            //绑定数据
        } else if (holder instanceof TwoTypeViewHolder) {
            //绑定数据
        } else if (holder instanceof ThreeTypeViewHolder) {
            //绑定数据
        }

    }

    @Override
    public int getItemCount() {
        return 100;
    }

    static class OneTypeViewHolder extends RecyclerView.ViewHolder {
        public OneTypeViewHolder(View itemView) {
            super(itemView);
        }
    }

    static class TwoTypeViewHolder extends RecyclerView.ViewHolder {
        public TwoTypeViewHolder(View itemView) {
            super(itemView);
        }
    }

    static class ThreeTypeViewHolder extends RecyclerView.ViewHolder {
        public ThreeTypeViewHolder(View itemView) {
            super(itemView);
        }
    }

}

```

>上面代码也没什么好说的，都是些RecyclerView的基本用法，但是上面的代码有很多弊端(多类型的 Adapter ).

传统实现方式的弊端:

1. 不利于后期的扩展，我们写代码时应该遵守"开闭原则"(知识点)，如果后期需要添加新的类型的 Item，这个时候我们就不得不修改  getItemType 方法来返回新的类型，并在 onCreateViewHolder 方法中添加对应的返回，最后还需要在 onBindViewHolder 中完成新Item的数据绑定。这里就涉及到几处代码的改动，要知道项目做到后期，尽量少修改原本写好的代码(巨坑)。
2. 耦合太大，在 onBindViewHolder 方法中每种类型的 ViewHolder 的数据绑定都在里面，完全无法复用。像上面所说的如果一旦我们修改，可能会出来各种莫名其妙的问题。

MultiType 的作用:
1. 如果后期需要添加新的类型，只需要实现对应的 ItemViewBinder 并调用 register 方法就可以完成加入。一行代码都不用修改。
2. 每种类型的 Item 的数据绑定都在对应的 ItemViewBinder 中，这样不同类型就不存耦合的问题，如果别的地方有相同的效果，还能直接复用。
3. MultiType 的实现一共只有十几个类,这个库仅仅只帮我们解耦了多类型Adapter的耦合问题。非常的简洁，和项目的耦合几乎为零。

通过上面的分析,MultiType的优点就体现出来了。

# MultiType的实现原理

上面我们说到了MultiType的类文件只有十几个，与其说MultiType是一个类库，还不如说MultiType是一个Android平台上多类型Adapter的解决方案(本人觉得很好的一个解决方案)，实现原理非常简单。下面我们就分析下核心的逻辑。

说到这里就有必要提下一些类似的实现方式，接入过网易云信的同学应该知道其中的消息界面的Adapter的实现方式就和MultiType类似，因为消息界面的消息类型繁多，如果逻辑都放在Adapter中，既不好维护也不好扩展。

## Talk is cheap，Show me the code。

``` java
//代码目录 : MultiTypeAdapter.java
//省略其余代码

//类型池
private TypePool typePool;

@Override
public final int getItemViewType(int position) {
    //断言
    assert items != null;
    //从 items 中取出数据对象
    Object item = items.get(position);
    //通过 indexInTypesOf 方法来返回对应的类型
    return indexInTypesOf(item);
}

@Override
public final ViewHolder onCreateViewHolder(ViewGroup parent, int indexViewType) {
    if (inflater == null) {
        inflater = LayoutInflater.from(parent.getContext());
    }
    ItemViewBinder<?, ?> binder = typePool.getItemViewBinders().get(indexViewType);
    binder.adapter = this;
    assert inflater != null;
    //调用对应 ItemViewBinder 类中的onCreateViewHolder
    return binder.onCreateViewHolder(inflater, parent);
}

@Override @SuppressWarnings("unchecked")
public final void onBindViewHolder(ViewHolder holder, int position, List<Object> payloads) {
    //断言
    assert items != null;
    //获取数据对象
    Object item = items.get(position);
    //获取 ItemViewBinder 对象 并调用 onBindViewHolder 方法来绑定数据
    ItemViewBinder binder = typePool.getItemViewBinders().get(holder.getItemViewType());
    binder.onBindViewHolder(holder, item, payloads);
}

//根据数据对象 获取对应的类型
int indexInTypesOf(@NonNull Object item) throws BinderNotFoundException {
    //获取对应类型的索引
    int index = typePool.firstIndexOf(item.getClass());
    if (index != -1) {
        @SuppressWarnings("unchecked")
        //通过索引去 TypePool 中获取 Linker。
        Linker<Object> linker = (Linker<Object>) typePool.getLinkers().get(index);
        //默认情况下 index 方法返回0 所以这里直接返回 上面获取到的 index。
        return index + linker.index(item);
    }
    throw new BinderNotFoundException(item.getClass());
}

//注册 ItemViewBinder
public <T> void register(@NonNull Class<? extends T> clazz, @NonNull ItemViewBinder<T, ?> binder) {
    //检测当前TypePool中是否已经有相同类型的，如果有就移除原有的。
    checkAndRemoveAllTypesIfNeed(clazz);
    typePool.register(clazz, binder, new DefaultLinker<T>());
}

```

>核心实现差不多就是上面这些，非常的简洁和简单。

# 总结
我认为在软件开发中，扩展 > 复用，网上有很多关于Adatper的封装，大部分都是简化代码(通过继承和多态实现)，但是在扩展这块做的还不是很好(个人观点)，所以还不如先解决扩展的问题在去优化重复的代码。当然我相信一个扩展非常好的代码复用也不会差到哪去。


