---
layout: post
title:  "LayoutInflate(布局填充器)的使用"
date:   6/3/2017 9:37:28 PM 
categories: Android
---
###1.初始化   

　　　● 通过Activity的方法  
		
		`LayoutInflater layoutInflater=mActivity.getLayoutInflater();`


　　　● 通过LayoutInflate的静态方法

		`LayoutInflater layoutInflater=LayoutInflater.from(mContext);`

　　　● 通过Context的方法  

		`LayoutInflater layoutInflater=mContext.getSystemService(Context.LAYOUT_INFLATER_SERVICE);`
     
       
###2.通过LayoutInflate将xml布局转化为View对象  
   
   　　主要是使用了LayoutInflate中的inflate()方法，我们经常在BaseAdapter的getView()方法中使用，主要利用其可以将xml文件转化为View的功能来将子条目的xml转化为View  
		
　　inflate()方法我们都不陌生，这里着重来看一下他的几个重载的方法：　　

　　　`inflate(@LayoutRes int resource, @Nullable ViewGroup root)`

　　　`inflate(XmlPullParser parser, @Nullable ViewGroup root)`　　

　　　`inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot)`

　　　`inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)`
     
　　我们一般使用的都是第一个和第三个方法，其实从源码中我们可以看出这四个方法最终都是调用了第四个方法，我们着重来看一下第四个方法　　

    //方法1
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root){
       return inflate(resource, root, root != null);//方法一其实 调用了方法三
    }

    //方法3
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                + Integer.toHexString(resource) + ")");
    }

    final XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);//方法三拿到解析器 调用了方法四
        } finally {
        parser.close();
    }
     }

    public View inflate(XmlPullParser parser, @Nullable ViewGroup root) 
    {
    return inflate(parser, root, root != null);//方法二调用了 方法四
    }
　　分析第四个方法的三个参数<br>

　　 Parser　　
        <p>　　通过我们的布局所产生的xml pull解析器的对象，也就是说inflate方法最终是通过Android的pull解析器将我们想要转化的xml文件进行解析，然后通过相应的节点以及相应的属性创建相应的View 从而达到xml布局转化为View的目的<br>

　　 Root　　　　　　　　　　　　　根视图<br> 

　　 attachToRoot　　　　　　　　　是否与根视图进行绑定<br>

　　<font color=blue>参数Parser：</font> 

　　inflate()最终是通过pull解析拿到相应的节点以及对应的属性 去创建相应的View我们可以在源码中找到他的相关创建流程 

　　　1.首先拿到了根标签的标签名以及标签的属性　


    final AttributeSet attrs = Xml.asAttributeSet(parser);
    final String name = parser.getName();

　　　2.	通过拿到的标签名以及相应的属性创建了根视图  
 
    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

　　　3.	继续进行xml解析 解析出所有的子View 并将其添加进temp视图
　　　

    rInflateChildren(parser, temp, attrs, true);

　　　4.	最终将视图进行返回 


　　<font color=blue>根视图 root-----连接到根视图  attachToRoot：</font> 

　　这个参数一般传入此View的父容器  比如itemView的父容器是ListView
  当然这个参数@Nullable是可以传null的，我们来看看源码中此参数对结果的影响

    if (root != null) {
        //------------------------和 root相关
        if (DEBUG) {
        System.out.println("Creating params from root: " +
                root);
        }
       // Create layout params that match root, if supplied
       // 此处有root的话 是会读取xml中的布局属性
       params = root.generateLayoutParams(attrs);
       // 如果传入值是false 则将此值设定到此View上
       if (!attachToRoot) { // 第三个参数为false  并且root不为null设置了 一个空的                          LayoutParams
        // Set the layout params for temp if we are not
        // attaching. (If we are, we use addView, below)
        
        temp.setLayoutParams(params);
       }
      // ------------------------和 root相关结束
    }

　　此处是用来判断 当root不为null时 就读取了xml文件的根布局的属性，然后再次判断 如果不关联到
根视图(if(!attachToRoot)) 则将此参数设置到temp视图的布局参数(LayoutParams)上


　　Ps：也就是说当 root(参数二)不为null 并且attachToRoot(参数三)为false 时 初始化了一个此 xml根标签的View 并且读取了此xml的根标签属性 对其进行了设置 (产生的View与xml文件中的参数一致)    
  
  　　<font color=red>总结一：当 root(参数二)不为null 并且attachToRoot(参数三)为false 时 初始化了一个此 xml根标签的View 并且读取了此xml的根标签属性 对其进行了设置 (产生的View与xml文件中的参数一致)</font>

　　那么当root不为null时 我们知道 会读取xml根布局的属性  当attachToRoot为true时 会是 情况？

　　继续来看:  

    if (root != null && attachToRoot) {
       //父View添加到了父View中
       root.addView(temp, params);
    }

  　　这个地方非常重要   当root不等于null 并且attachToRoot 等于true时 我们可以看到 root直接将此View添加到了root的容器中 并且携带此View的xml中解析的参数params,还有这是一个方面，我们回到此方法刚开始的地方可以看到:

    View result = root;  

　　然后在方法的最后一句可以看到return 出去的View是

    return result;

　　这样一看怎么是直接将父容器进行返回了？ 和平时用的怎么不一样  我们来看看在这两句中间可以看到:

    if (root == null || !attachToRoot) {
         //TODO 将根视图给了  result
          result = temp;
     }

　　也就是换句话说  只有当root不等于null并且attachToRoot 等于true时 返回 root视图  其他的情况 返回的是当前xml所表示的视图

　　<font color=red>总结二：当 root(参数二)不为null 并且attachToRoot(参数三)为true 时 初始化了一个此 xml根标签的View 并且读取了此xml的根标签属性    并且将此View放入了根视图(root)中 并且同时携带了参数(params) ,注意一点 此种情况下 返回的是根视图root 并不是我们想要转换的xml所表示的视图)</font>

　　前面两种说了 当 root不为null的情况 当root为null的情况下 只是对View进行了初始化 
至于View的LayoutParams 并没有像前面两种一样得到设置

　　<font color=red>总结三：当root为 null时   attachToRoot为false 时 所得到的是当前xml中的View  但此View的LayoutParams并没有像之前的两种一样进行设置，所以无论根布局中的View宽高如何 都是无效的
新产生的View并没有将其携带 新产生的View则是 默认的宽高<br>　　其实当root为null时  我们仔细查看源码会发现attachToRoot 参数并未产生用途 ，所以和上面的情况相同</font>


 **一般情况下 我们在使用两个参数的方法时 其实是调用了三个参数的方法:**

    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
               return inflate(resource, root, root != null);
    }

**当传入 布局id 以及 null时   其实是调用了三个参数的 int null false 同总结三**

**当传入 布局id 以及 非null时   其实是调用了三个参数的 int 非null true 同总结二**

　




　　
