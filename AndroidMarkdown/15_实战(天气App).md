# 天气预报App

## 功能需求和技术可行性分析

### 功能需求
- 可以搜索全球大多数国家的各城市数据
- 可以查看全球绝大多数城市的天气信息
- 可以自由切换城市，查看其它城市的天气
- 可以手动刷新实时的天气
- 可以对标一个优秀的天气APP-----彩云天气

### 技术可行性分析
先不谈我自己有多拉跨
- 彩云天气的免费API接口可以查看全球100多个国家的城市数据
- API每天最多会提供1万次的免费请求

### 获取彩云天气开放API


## git项目到github


## 搭建MVVM项目架构
- **M:** Model是数据模型部分
- **V:** View是界面展示部分
- **VM:** 连接数据模型和界面展示的桥梁，从而实现业务逻辑和界面展示分离的程序结构设计
- 一个优秀的项目结构除了这三部分，还有仓库，数据源等等

[![qnAn5n.png](https://s1.ax1x.com/2022/03/21/qnAn5n.png)](https://imgtu.com/i/qnAn5n)
- **Ui控制层**包含了我们平时写的Activity，Fragment，布局文件等等
- **ViewModel层**用于持有和UI元素有关的数据，以保证这些数据在屏幕选装的时候不会丢失，并且还要提供接口给UI控制层调用，以及和仓库层通信
- **仓库层**主要是判断调用方请求的数据应该是从本地数据源获取还是从网络数据源获取，并将获取到的数据返回到调用方
- **本地数据源**可以使用数据库，文件，SharedPerferences实现
- **网络数据源**则通常是Retrofit访问服务器提供的WebService接口实现
- 所有箭头都是单向的，表示上层持有下层的引用，并且引用不能跨层，就是第一层不会持有第三层的引用

### 准备工作
[![qnVrAH.png](https://s1.ax1x.com/2022/03/21/qnVrAH.png)](https://imgtu.com/i/qnVrAH)
- **logic**: 业务逻辑相关代码
  - **dao**：存放数据访问对象相关代码
  - **model**: 对象模型相关代码
  - **network**: 网络相关代码
- **ui**：界面展示相关代码
  - **place**：天气中的位置主界面
  - **weather**: 天气中的天气主界面

**导包**
```kotlin
dependencies {
    ···
    implementation 'androidx.recyclerview:recyclerview:1.2.0'
    implementation 'androidx.lifecycle:lifecycle-extensions:2.2.0'
    implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.3.1'
    implementation 'com.google.android.material:material:1.3.0'
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.6.1'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.6.0-native-mt'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.6.0-native-mt'
}
```

## 搜索全球城市数据

### 实现逻辑层代码

**由于MVVM架构的设计，ViewModel层开始就不再持有Activity的引用，所以我们需要全局获取context**
```kotlin
class BoWeatherApplication:Application() {
    companion object{
        const val TOKEN=""//唯一的彩云天气的令牌，获取数据用
        //伴生匿名单例类，也就是说一个application实例中只会有一个这个类，就让整个APP只会有一个全局Context
        @SuppressLint("StaticFieldLeak")//把context注解成静态的
        lateinit var context: Context
    }

    override fun onCreate() {
        super.onCreate()
        context=applicationContext//设置Context
    }
}
```
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.bo.boweather.android">
    <!--android:name=".BoWeatherApplication这个是告知程序启动的时候要初始化BoWeatherApplication类,从而获取全局Context-->
    <application
        android:name=".BoWeatherApplication"
        ···>
        ···
    </application>
</manifest>
```

#### 按照MVVM项目结构图自底向上实现

**定义位置数据模型**
```kotlin
data class PlaceResponse(val status:String,val places:List<Place>) {
    /***
     * status:请求的状态
     * places:位置
     */
}

data class Place(val name: String, val location:Location,
                 @SerializedName("formatted_address") val address:String){
    /***
     * name:位置名称
     * location:位置的经纬度
     * @SerializedName("formatted_address") address:位置的地址,
     * 可能请求返回的字段json命名和kotlin命名规范不一样，用注解的方式给json字段和kotlin字段建立一个映射关系
     */

}

data class Location(val lng:String,val lat:String){
    /***
     * lng:位置的经度
     * lat:位置的纬度
     */
}
```

**网络层**

**访问彩云天气城市搜索API的Retrofit接口**
```kotlin
interface PlaceService {
    @GET("v2/place?token=${BoWeatherApplication.TOKEN}&lang=zh_CN")
    fun searchPlaces(@Query("query") query: String): Call<PlaceResponse>
    /***
     * 当有类实现接口调用searchPlaces的时候Retrofit会自动发起一条请求，去访问@GET中的路径
     * 搜索的参数中只有query是动态指定的，有@Query注解自动构建到@GET的路径中
     * Call<PlaceResponse>就会把服务器返回的json数据解析成PlaceResponse
     */
}
```

**为了使用接口，还需要创建一个Retrofit构建器**
```kotlin
object ServiceCreator {
    private const val BASE_URL="https://api.caiyunapp.com/"

    private val retrofit= Retrofit.Builder()
        .baseUrl(BASE_URL)//根路径
        .addConverterFactory(GsonConverterFactory.create())//指定解析时的转换库(gson解析json字符串)
        .build()

    fun<T> create(serviceClass: Class<T>):T= retrofit.create(serviceClass)//创建接口的动态代理对象

    inline fun <reified T> create():T= create(T::class.java)//内联函数+泛型实化创建接口的动态代理对象
}
```

**定义一个同一的网络数据源访问入口，对所有网络请求的API进行封装**
```kotlin
object BoWeatherNetwork {
    private val placeService=ServiceCreator.create(PlaceService::class.java)
    //利用ServiceCreator这个单例类(看作静态)创建出一个placeService的动态代理对象

    suspend fun searchPlaces(query:String)= placeService.searchPlaces(query).await()
    //发起搜索城市的请求，searchPlaces是一个挂起函数
    //暂停函数是可以控制启动，暂停和恢复函数。
    // 关于挂起函数要记住的最重要的一点是，仅允许从协程或另一个挂起函数。
    //这个函数的流程就是:
    //发起网络请求之后await就会把当前的协程阻塞，searchPlaces就会挂起，当await中获取到数据之后就会恢复当前协程的执行


    private suspend fun <T> Call<T>.await():T{
        return suspendCoroutine { continuation ->
            enqueue(object :Callback<T>{
                override fun onResponse(call: Call<T>, response: Response<T>) {
                    val body=response.body()
                    if(body!=null) continuation.resume(body)
                    else continuation.resumeWithException(
                        RuntimeException("response body is null")
                    )
                }

                override fun onFailure(call: Call<T>, t: Throwable) {
                    continuation.resumeWithException(t)
                }
            })
        }
    }
}
```

**没有进行本地缓存，直接每次都网络请求获取最新的城市数据，仓库层的同一封装入口**
```kotlin
object Repository {
    /*liveData()函数是lifecycle-livedata-ktx库中提供的一个功能,他能自动构建一并返回一个liveData对象,
    然后在他的代码块中提供一个挂起函数的上下文，这样我们就可以在liveData()函数中调用任意的挂起函数*/
    fun searchPlaces(query:String)= liveData(Dispatchers.IO) {
        //由于网络请求是无法在主线程中运行的，所以我们用Dispatchers.IO把liveData代码块中的逻辑求换到子线程中运行
        val result=try {
            val placeResponse=BoWeatherNetwork.searchPlaces(query)//发起请求接收数据
            if(placeResponse.status=="ok"){//如果请求的结果是成功的话
                val places=placeResponse.places///就把请求的结果数据保存下来
                Result.success(places)//把成功获取的城市数据列表包装起来
            }else{
                Result.failure(RuntimeException("response status is ${placeResponse.status}"))//如果没成功就把异常信息包装起来
            }
        }catch (e:Exception){
            Result.failure<List<Place>>(e)//如果没成功就把异常信息包装起来
        }
        emit(result)//类似调用LiveData.setValue()，通知数据变化相当于把result设置到livedata中
    }
}
```

**逻辑层的最后一步-----ViewModel层**
- ViewModel相当于逻辑层和UI层之间的一个桥梁
- ViewModel通常和Activity或Fragment一一对应
  
```kotlin
class PlaceViewModel:ViewModel() {
    private val searchLiveData=MutableLiveData<String>()//存放的字符串是城市的名称

    val placeList=ArrayList<Place>()//用于缓存在界面上显示的城市信息

    val placeLiveData=Transformations.switchMap(searchLiveData){query->
        Repository.searchPlaces(query)//根据城市的名称来查询
    }

    fun searchPlaces(query:String){
        searchLiveData.value=query
    }
}
```

### UI层

**搜索城市数据的功能要复用，所以写在Fragmnet中，要复用的时候就直接在布局中引入Fragment**
```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="?android:windowBackground">

    <ImageView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:id="@+id/bgImageView"
        android:layout_alignParentBottom="true"
        android:src="@drawable/bg_place"/>

    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="60dp"
        android:id="@+id/actionBarLayout"
        android:background="@color/purple_200">

        <EditText
            android:layout_width="match_parent"
            android:layout_height="40dp"
            android:id="@+id/searchPlaceEdit"
            android:layout_gravity="center_vertical"
            android:layout_marginStart="10dp"
            android:layout_marginEnd="10dp"
            android:paddingStart="10dp"
            android:paddingEnd="10dp"
            android:hint="输入地址"
            android:background="@drawable/search_bg"/>
    </FrameLayout>

    <androidx.recyclerview.widget.RecyclerView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/recyclerView"
        android:layout_below="@+id/actionBarLayout"
        android:visibility="gone"/>
</RelativeLayout>
```

**列表用的是recyclerview，所以需要他的子项布局,用的是卡片式布局**
```xml
<com.google.android.material.card.MaterialCardView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="130dp"
    android:layout_margin="12dp"
    app:cardCornerRadius="4dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:layout_gravity="center_vertical"
        android:layout_margin="18dp">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/placeName"
            android:textColor="?android:attr/textColorPrimary"
            android:textSize="20sp"/>
        <!--用于显示搜索到的地区名-->

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/placeAddress"
            android:layout_marginTop="10dp"
            android:textColor="?android:attr/textColorSecondary"
            android:textSize="14sp"/>
        <!--显示该地区的详细地址-->
    </LinearLayout>

</com.google.android.material.card.MaterialCardView>
```

**recyclerview是需要适配器的**
```kotlin
class PlaceAdapter(private val fragment: Fragment, private val placeList: List<Place>):
    RecyclerView.Adapter<PlaceAdapter.ViewHolder>(){

    inner class ViewHolder(view:View):RecyclerView.ViewHolder(view){
        val placeName:TextView=view.findViewById(R.id.placeName)
        val placeAddress:TextView=view.findViewById(R.id.placeAddress)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view=LayoutInflater.from(parent.context).inflate(R.layout.place_item,parent,false)
        return ViewHolder(view)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val place=placeList[position]
        holder.placeAddress.text=place.address
        holder.placeName.text=place.name
    }

    override fun getItemCount(): Int {
        return placeList.size
    }
}
```

**实现Fragment**
```kotlin
class PlaceFragment : Fragment() {

    private val viewModel by lazy { ViewModelProviders.of(this).get(PlaceViewModel::class.java) }

    private lateinit var adapter: PlaceAdapter

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        return inflater.inflate(R.layout.fragment_place, container, false)
    }

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)

        val layoutManager = LinearLayoutManager(activity)
        recyclerView.layoutManager = layoutManager
        adapter = PlaceAdapter(this, viewModel.placeList)
        recyclerView.adapter = adapter
        searchPlaceEdit.addTextChangedListener { editable ->
            val content = editable.toString()
            if(content.isNotEmpty()) {//当搜索框中的内容发生变化的时候就获取新的内容
                viewModel.searchPlaces(content)
            }else {
                recyclerView.visibility = View.GONE
                bgImageView.visibility = View.VISIBLE
                viewModel.placeList.clear()
                adapter.notifyDataSetChanged()
            }
        }
        viewModel.placeLiveData.observe(this, Observer{ result ->
            //监控ViewData中的数据变化,当数据有变化就回调到observe中
            val places = result.getOrNull()
            if (places != null) {
                recyclerView.visibility = View.VISIBLE
                bgImageView.visibility = View.GONE
                viewModel.placeList.clear()
                viewModel.placeList.addAll(places)//把地址添加到placeList中后适配器就会感应到，然后把数据添加到UI中
                adapter.notifyDataSetChanged()//刷新UI
            }else{
                Toast.makeText(activity,"未能查询到任何地点",Toast.LENGTH_LONG).show()
                result.exceptionOrNull()?.printStackTrace()//数据为空就toast一个提示
            }
        })
    }

}
```

**Fragment是不能直接显示在界面上的，需要添加到Activity中才行**
```xml
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <fragment
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/placeFragment"
        android:name="com.bo.boweather.android.ui.place.PlaceFragment"/>

</FrameLayout>
```

**PlaceFragment中已经定义了搜索框，那么标题栏就不要**
```xml
<resources xmlns:tools="http://schemas.android.com/tools">
    <!-- Base application theme. -->
    <style name="Theme.随波天气" parent="Theme.MaterialComponents.DayNight.NoActionBar">
        ···
    </style>
</resources>
```

**添加网络权限**
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.bo.boweather.android">
    <uses-permission android:name="android.permission.INTERNET"/>
    ···
</manifest>
```
