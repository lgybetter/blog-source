---
title: 安卓自定义控件RecyclerView
date: 2016-7-31 7:56:41
tags:
  - 安卓
categories:
  - 客户端
---

> RecyclerView是安卓5.0版本对列表视图控件ListView的升级，也是属于MD设计的一部分，其功能十分强大，但是使用起来相比ListView比较复杂。


## 布局文件

在xml布局文件中添加RecyclerView控件

```xml
<android.support.design.widget.CoordinatorLayout
  xmlns:android="http://schemas.android.com/apk/res/android"
  android:layout_width="match_parent"
  android:layout_height="match_parent"
  android:fitsSystemWindows="true"
  android:orientation="vertical">
  <RelativeLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <android.support.v7.widget.RecyclerView
      android:id="@+id/rv_message_list"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      android:layout_below="@+id/view" />
  </RelativeLayout>
  <android.support.design.widget.FloatingActionButton
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="right|bottom"
    android:layout_margin="@dimen/fab_margin"
    android:src="@android:drawable/ic_dialog_email" />
</android.support.design.widget.CoordinatorLayout>
```

## JAVA文件中使用控件的步骤

### 定义适配器：

使用自定义的适配器，继承了RecyclerView.Adapter，并给出封装的Holder类型

```java
public class MessageAdapter extends RecyclerView.Adapter<MessageViewHolder> {
  //上下文引入
  private Context context;
  //保存对象的信息列表
  private List<MessageBeanClass> dataList;
  //用于进行点击事件的监听器
  private MyItemClickListener mItemClickListener;
  private MyItemLongClickListener mItemLongClickListener;
  //构造函数，传递上下文和数据列表
  public MessageAdapter(Context c, List<MessageBeanClass> d) {
    context = c;
    dataList = d;
  }
  //创建Holder
  @Override
  public MessageViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    MessageViewHolder messageViewHolder = (MessageViewHolder) parent.getTag();
    if(messageViewHolder == null) {
      messageViewHolder = new MessageViewHolder(LayoutInflater.
      from(context).inflate(R.layout.activity_tab_message_item,parent,false),
      mItemClickListener,mItemLongClickListener);
    }
    return messageViewHolder;
  }
  //对每个列表中的选项进行设置
  @Override
  public void onBindViewHolder(MessageViewHolder holder, int position) {
    holder.name.setText(dataList.get(position).getName());
    holder.message.setText(dataList.get(position).getMessageText());
    holder.time.setText(dataList.get(position).getTime());
  }
  //返回列表的长度用于显示列表中的选项数目
  @Override
  public int getItemCount() {
    return dataList.size();
  }
  //监听器设置
  public void setOnItemClickListener(MyItemClickListener listener){
    this.mItemClickListener = listener;
  }
  public void setOnItemLongClickListener(MyItemLongClickListener listener){
    this.mItemLongClickListener = listener;
  }
}
```

### 定义ViewHolder

```java
//自定义的Holder,继承RecyclerView.ViewHolder,引用点击事件接口
public  class  MessageViewHolder  extends  RecyclerView.ViewHolder implements View.OnClickListener,
  View.OnLongClickListener
  {
    //事件监听器
    private MyItemClickListener mListener;
    private MyItemLongClickListener mLongListener;
    public TextView name, message,time;
    public MessageViewHolder(View itemView,MyItemClickListener 
    listener,MyItemLongClickListener longListener) {
    super(itemView);
    this.mListener = listener;
    this.mLongListener = longListener;
    initView();
  }

  private void initView() {
    time = (TextView) itemView.findViewById(R.id.tv_time);
    name = (TextView) itemView.findViewById(R.id.tv_user_name);
    message = (TextView) itemView.findViewById(R.id.tv_user_message);
    itemView.setBackgroundResource(R.drawable.recycler_view_background);
    itemView.setOnClickListener(this);
    itemView.setOnLongClickListener(this);
  }
  @Override
  public void onClick(View view) {
    if(mListener != null) {
      mListener.onItemClick(view,getAdapterPosition());
    }
  }
  @Override
  public boolean onLongClick(View view) {
    if(mLongListener != null) {
      mLongListener.onItemLongClick(view,getAdapterPosition());
    }
    return true;
  }
}
```

### 在Activity或Fragment中使用：

```java
public class MessageFragment extends Fragment implements MyItemClickListener, 
  MyItemLongClickListener{
  /**
    * 测试数据
    */
  private String[] nameTest = {"TOM","BOB","JAME"};
  private String[] messageTest = {"FUCK YOU","YOU GET OUT","ONE NIGHT"};
  private String[] timeTest = {"2小时前","昨天","2016-12-31"};
  private RecyclerView recyclerView;
  private View messageView;
  private MessageAdapter adapter;
  private MessageBeanClass messageBean;
  private List<MessageBeanClass> messageList = new ArrayList<>();

  @Override
  public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
    super.onCreateView(inflater, container, savedInstanceState);
    messageView = inflater.inflate(R.layout.activity_tab_message, container,false);
    initView();
    initData();
    return messageView;
  }

  private void initData() {
  messageList.clear();
    for(int i = 0; i < 3 ; i++) {
      //Bitmap bitmap = BitmapFactory.decodeResource(getResources(),imageTest[i]);
      String name = nameTest[i];
      String message = messageTest[i];
      String time = timeTest[i];
      messageBean = new MessageBeanClass(name,message,time,0);
      messageList.add(messageBean);
    }
    adapter = new MessageAdapter(getContext(),messageList);
    recyclerView.setAdapter(adapter);
    adapter.setOnItemClickListener(this);
    adapter.setOnItemLongClickListener(this);
  }

  private void initView() {
    recyclerView = (RecyclerView) messageView.findViewById(R.id.rv_message_list);
    //设置RecyclerView的布局管理器，如果是网格类型可以选择GridLayoutManager
    recyclerView.setLayoutManager(new LinearLayoutManager(getContext()));
    //给视图选项添加分割线
    recyclerView.addItemDecoration(newDividerItemDecoration(getContext(),
    DividerItemDecoration.VERTICAL_LIST));
  }

  @Override
  public void onActivityCreated(Bundle savedInstanceState){
    super.onActivityCreated(savedInstanceState);
  }

  @Override
  public void onItemClick(View view, int postion) {
    MessageBeanClass messageBeanClass = messageList.get(postion);
    if(messageBeanClass != null) {
      Intent intent = new Intent();
      intent.setClass(getContext(), ChattingActivity.class);
      startActivity(intent);
    }
  }

  @Override
  public void onItemLongClick(View view, int postion) {
    MessageBeanClass messageBeanClass = messageList.get(postion);
      if(messageBeanClass != null) {
      Toast.makeText(getContext(),"onItemLongClick"postion,Toast.LENGTH_SHORT).show();
    }
  }
}
```

### 点击事件的接口：

```java
public interface MyItemClickListener {
  public void onItemClick(View view, int postion);
}

public interface MyItemLongClickListener {
  public void onItemLongClick(View view, int postion);
}
```

于是这样就完成了对RecyclerView的使用，由于在上述例子中使用了分割线，但是因为绘制分割线的代码较长，而且对该实例中可以不使用，于是就省略代码，避免篇幅太长了。

