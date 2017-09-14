---
layout: post
title:  "ListView控件的使用和优化整理"
date:   2016-1-7
categories: listview android viewHolder adapter
author: 方浩然
header-img: "img/android.jpg"
---


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因为数据结构的课程设计想要用android实现最短路径算法，做个小的应用，so..最近又开始看安卓了。到`ListView`控件那一章，之前写过的基本都忘记了，花了比较长的时间重新看书。发现了很多细节问题，这些问题都是第一遍看的时候没有仔细想过的，所以记录下来，可能在以后的某个时候再重新看的时候，又会有不同的理解。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ListView`实现了一个可以滑动的列表，应用极其广泛，几乎所有的app都会用它。比如微博主界面的一条条消息，手机通讯录，微信好友也是一条条地显示。几乎需要大规模呈现单一性质的数据的地方都会使用ListView。ListView的使用也还是比较方便的。代码如下。

```java
listView = (ListView) findViewById(R.id.list_view);         //找到listview
initFruitList();                                            //初始化List数据表
FruitAdapter adapter = new FruitAdapter(this,R.layout.fruit_item,myFuitList);  //新建一个适配器
listView.setAdapter(adapter);                               //为listView设置适配器
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;首先根据id找到`资源文件`中的ListView，初始化需要填充的数据，我的例子中写的是一个水果类，很简单，包括`水果图的id`和`水果的名称`，形成一个List。接着通过继承`ArrayAdapter`来建个自己的`适配器`,最后为listview设置适配器，通过适配器将List数据和ListView连接起来，每个listview的`item`放一个数据，这样一个listView就完成了。不过有个比较严重的问题，数据多了，滑动的时候就会出现卡顿.

~~~java
public class FruitAdapter extends ArrayAdapter<Fruit> {
    private int resourceId;         //item布局的id
    //Adapter构造器
    public FruitAdapter(Context context, int resourceId, List<Fruit> objects){
        super(context,resourceId,objects);
        this.resourceId = resourceId;
    }

    //加载每次滑动的item布局
    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        Fruit fruit = getItem(position);            //当前滑动的item水果实例
        View view;									//当前需要拿到的view
        view = LayoutInflater.from(getContext()).inflate(resourceId, null);      //通过id从xml文件里解析到layout  也就是fruit_item
        ImageView imageView = (ImageView) view.findViewById(R.id.fruit_image);	//view里面find
      	TextView textView = (TextView)view.findViewById(R.id.fruit_name);
        imageView.setImageResource(fruit.imageId);      //设置当前item上的图像
        textView.setText(fruit.getName());              //设置当前item上的水果名
        return view;
    }

}   //FruitAdapter
~~~

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对FruitAdapter适配器构造重点在重写父类`ArrayAdapter`中`getView`方法。通过阅读API，了解到这个方法的主要作用是得到当前position的View，在上下反复滑动屏幕的过程中getView是一直不断地被调用的。ListView中每个Item的每一次呈现都要回调一次getView方法。这就不难得出为什么在上下滑动的时候会出现卡顿现象。代码中一直在通过xml布局文件新建一个`view`，然后不停的`findViewById`来找布局中的图片和文字，并且为他们设置当前Fruit对象的相关成员。然后返回这个新建的`view`。从代码里很容易看出来，每次从布局文件中inflate出view和findview的过程是重复无意义的。于是优化的小技巧就出现了。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用getview参数中的`convertView`作缓存，用来保存view，构造一个内部类`ViewHolder`，来保存item需要每次从xml解析出来的ImageView和TextView，顺理成章地把这个viewHolder放在convertView缓存里(`setTag`)。每次滑动需要display当前item的view的时候，检查缓存里有没有，如果有，那么直接使用，并且从convertView中(`getTAG`)得到`ImageView`和`TextView`，最后为每个item设置图片(似乎也是重复的)。因为android对于listView的展现就是这样重复地调用getView，所以最后这一步似乎不可以再优化了？代码如下:

~~~java
@Override
    public View getView(int position, View convertView, ViewGroup parent) {
        Fruit fruit = getItem(position);            //当前滑动的item
        View view;
        ViewHolder viewHolder;
        if(convertView == null) {					//缓存为空
            view = LayoutInflater.from(getContext()).inflate(resourceId, null);     //找到fruit_item layout
            viewHolder = new ViewHolder();				//新建viewHolder
            viewHolder.imageView = (ImageView)view.findViewById(R.id.fruit_image);
            viewHolder.textView = (TextView)view.findViewById(R.id.fruit_name);
            view.setTag(viewHolder);					//存到convertView中
        } else {
            view = convertView;										//从缓存拿到view和view中的文字 图片
            viewHolder = (ViewHolder)view.getTag();
        }

        viewHolder.imageView.setImageResource(fruit.getImageId());      //设置当前item上的图像
        viewHolder.textView.setText(fruit.getName());              //设置当前item上的水果名
        return view;
    }

    class ViewHolder {						//内部 viewHolder类
        ImageView imageView ;
        TextView textView;
    }
}   //FruitAdapter

~~~

End
