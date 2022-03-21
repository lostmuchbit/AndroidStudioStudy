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
    private const val BASE_URL="http://api.caiyunapp.com"

    private val retrofit= Retrofit.Builder()
        .baseUrl(BASE_URL)//根路径
        .addConverterFactory(GsonConverterFactory.create())//指定解析时的转换库(gson解析json字符串)
        .build()

    fun<T> create(serviceClass: Class<T>):T= retrofit.create(serviceClass)//创建接口的动态代理对象

    inline fun <reified T> create():T= create(T::class.java)//内联函数+泛型实化创建接口的动态代理对象
}
```
