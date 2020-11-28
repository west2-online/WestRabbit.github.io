---
layout:      post
title:      "Gocv--Go语言调用Opencv4"
date:       2020-11-28
author:     "林怀颖"
catalog:     true
tags:
     - 19级
     - Android
---

**

## 关于安卓的RecycleView的使用
首先导入RecyclerView组件，这里我已经添加了，没添加的话旁边会有个下载的按钮，点击下载即可
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201018111519289.png#pic_center)

***在acticity_main.xml布局文件中加入RecyclerView组件***

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"

    >
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/id_recyclerview"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        />
</RelativeLayout>
```



***然后定义recycler_item.xml布局文件*

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal"
    android:layout_marginTop="10dp"
    android:onClick="myClick">
    <RelativeLayout
        android:id="@+id/linear_btu"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical" >
        <ImageView
            android:id="@+id/iv"
            android:layout_width="80dp"
            android:layout_height="60dp"
            />
        <TextView
            android:id="@+id/number"
            android:layout_width="80dp"
            android:layout_height="60dp"
            android:gravity="center"
            android:textSize="20dp"
            android:text="1"
            android:textColor="#FFFFFF"/>
    </RelativeLayout>

    <RelativeLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginLeft="10dp"
        android:layout_marginTop="0dp">

    <TextView
        android:id="@+id/name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="第一章 Android基础入门"
        android:textSize="20sp"

        />

    <TextView
        android:id="@+id/introduce"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="22dp"
        android:ellipsize="end"
        android:maxLines="2"
        android:text="共计5题"
        android:textSize="16sp" />
    </RelativeLayout>
</LinearLayout>
```

*接下来就是Activity部分*





```
public class RecyclerActivity extends AppCompatActivity {
    private String[] names={"第1章 Android基础入门","第2章 Android常见界面布局","第3章 Android常见界面布局","第4章 程序活动单元Activity"
    ,"第5章 数据存储","第6章 内容提供者","第7章 广播机制","第8章 服务","第9章 网络编程","第10章 综合项目"};
private int[] icons={R.drawable.exercises_bg_1,R.drawable.exercises_bg_2,R.drawable.exercises_bg_3,R.drawable.exercises_bg_4,
        R.drawable.exercises_bg_1,R.drawable.exercises_bg_2,R.drawable.exercises_bg_3,R.drawable.exercises_bg_4,
        R.drawable.exercises_bg_1,R.drawable.exercises_bg_2};
private  String[] introduces={"共计5题","共计5题","共计5题","共计5题","共计5题","共计5题","共计5题","共计5题","共计5题","共计5题"};
private  String[] numbers={"1","2","3","4","5","6","7","8","9","10"};
    RecyclerView mRecyclerView;
    HomeAdapter mAdapter;
    @RequiresApi(api = Build.VERSION_CODES.O)
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
         mRecyclerView = findViewById(R.id.id_recyclerview);
        mRecyclerView.setLayoutManager(new LinearLayoutManager(this));
        mAdapter = new HomeAdapter();
        mRecyclerView.setAdapter(mAdapter);
        setTitle("                      移动开发设计习题");

    }
    public void myClick(View view)
    {
        int position = mRecyclerView.getChildAdapterPosition(view);
       Toast.makeText(RecyclerActivity.this,names[position],Toast.LENGTH_LONG).show();
    }
class HomeAdapter extends RecyclerView.Adapter<HomeAdapter.MyViewHolder> {

    @NonNull
    @Override
    public MyViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {

        return new MyViewHolder(LayoutInflater.from(RecyclerActivity.this).inflate(R.layout.recyler_item, parent, false));
    }

    public void onBindViewHolder(MyViewHolder holder, int position) {
        holder.name.setText(names[position]);
        holder.iv.setImageResource(icons[position]);
        holder.indroduce.setText(introduces[position]);
        holder.number.setText(numbers[position]);
    }

    public int getItemCount() {
        return names.length;
    }

    class MyViewHolder extends RecyclerView.ViewHolder {
        TextView name;
        ImageView iv;
        TextView indroduce;
       TextView number;
        MyViewHolder(View view) {
            super(view);
            name = view.findViewById(R.id.name);
            iv =  view.findViewById(R.id.iv);
            indroduce = view.findViewById(R.id.introduce);
            number=view.findViewById(R.id.number);

        }
    }
}

}
```
效果图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020101811201168.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpbmUxMjM0NTU=,size_16,color_FFFFFF,t_70#pic_center)



