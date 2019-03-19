---
title: retrofit2+okhttp3+rxjava2的网络框架整理
date: 2018-10-07 21:00:00
tags: 
    - Android
    - 总结
---
[TOC]

## 期望
- 不需要业务层处理code-msg-data的顶层返回封装格式.错误的msg和网络错误一起走错误回调.
- 发送请求前能检查网络,网络不可用时直接返回错误
- 不需要每次手动添加通用参数token
- token过期,可以重试1次,重试仍然失败就退出登录
- token冲突,要自动退出登录
- 业务层调用尽量要简单

最终效果
```Java
    CommonRequest.getInstance()
                .sendRequest(CommonRequest.getInstanceService().login("username","123456"))
                .subscribe(new BaseObserver<String>() {

                    @Override
                    protected void onHandleSuccess(String s) {
                        token = s;
                    }

                    @Override
                    protected void onHandleError(String pError) {

                    }
                });
```


## 实现
### Service
retrofit2里的Service,主要负责接口的定义

```java
public interface CommonService {
    @POST("user/login")
    Observable<BaseJsonResult<String>> login(@Query("username") String pUsername,
                                             @Query("password") String pPassword);
                                             
    @GET("camera/getInfoList")
    Observable<BaseJsonResult<BaseJsonPageResult<InfoBean>>> getInfoList(@Query("keyword") String pKeyword,
                                                                        @Query("page") int page,
                                                                        @Query("pageSize") int pageSize);
}
```
### RetrofitFactory
负责Retrofi和OKHttpClient初始化配置

```java
public class RetrofitFactory {
    private static String TAG = "RetrofitFactory";

    //初始化retrofit
    public static Retrofit getRetrofit(OkHttpClient okHttpClient, String baseUrl) {

        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(baseUrl)
                .client(okHttpClient)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();
        return retrofit;
    }

    //初始化OkHttpClient
    public static OkHttpClient getOkHttpClient(boolean isHttps, UrlInterceptor urlInterceptor) {
        //设置Http请求拦截器
        File httpCacheDirectory = new File(MyApplication.getInstance().getCacheDir(), "retrofit");
        Cache cache = new Cache(httpCacheDirectory, 10);
        HttpLoggingInterceptor httpLoggingInterceptor = new HttpLoggingInterceptor();
        httpLoggingInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY);
        OkHttpClient.Builder builder = new OkHttpClient.Builder()
                .connectTimeout(10, TimeUnit.SECONDS)
                .readTimeout(10, TimeUnit.SECONDS)
                .writeTimeout(10, TimeUnit.SECONDS)
                //  .retryOnConnectionFailure(false)
                .cache(cache)
                .addInterceptor(getNewWorkStateIntercepter())
                .addInterceptor(getParameterIntercepter())
                .addInterceptor(httpLoggingInterceptor);

        if (urlInterceptor != null) {
            builder.addInterceptor(urlInterceptor);
        }

        OkHttpClient httpClient = builder.build();
        return httpClient;
    }
    
    //初始化CommonService
     public static CommonService getCommonService(String pApiServerIp, String pApiServerPort, String pUrlPre) {
        CommonService service;
        String ipAddress = pApiServerIp;
        String port = pApiServerPort;
        String baseUrl = "http://" + ipAddress + ":" + port + "/";
        if (!TextUtils.isEmpty(pUrlPre.trim())) {
            baseUrl += pUrlPre + "/";
        }
        Retrofit retrofit = RetrofitFactory.getRetrofit(getOkHttpClient(false, null), baseUrl);
        service = retrofit.create(CommonService.class);
        return service;
    }
}
```
#### NewWorkStateIntercepter
在发送前获取网络状态,当网络状态不可用(若想控制WiFi流量拦截也在这里)时,直接返回error

```java
  private static Interceptor getNewWorkStateIntercepter() {
        return chain -> {
            if (NetworkUtils.isConnected(MyApplication.getInstance())) {
                return chain.proceed(chain.request());
            } else {
                throw new IOException("没有网络连接,请检查网络状态");
            }
        };
    }
```
#### ParameterIntercepter
自动添加通用参数token

```Java
 private static Interceptor getParameterIntercepter() {
        return chain -> {
            Request request = chain.request();
            //登录请求不需要添加token
            if (MTextUtils.isEmpty(MyApplication.getInstance().getToken())) {
                return chain.proceed(request);
            }
            HttpUrl url = request.url();
            HttpUrl newUrl = url.newBuilder()
                    .addQueryParameter("token", MyApplication.getInstance().getToken())
                    .build();
            request = request.newBuilder()
                    .url(newUrl)
                    .addHeader("Connection", "close")
                    .build()
            ;

            Response response = chain.proceed(request);
            return response;
        };
    }
```
### CommonRequest
- 全局单例
- 初始化网络请求的地方,设置和缓存请求地址的值
- 发送请求的地方

```
   public Observable sendRequest(Observable observable) {
        return observable
                .compose(RxSchedulers.io_main())//线程转换,我写了另外一个方法,回调固定在子线程的.也可以用bool判断合并成一个函数
                .flatMap((Function<BaseJsonResult, Observable<? extends BaseJsonResult>>) response -> {
                    if (response.isTokenError()) {// 判断token是否过期
                        MyApplication.getInstance().setToken(null);
                        return Observable.error(new Throwable(TokenRefresher.TOKEN_NEEDRETRY));
                    } else if (response.isTokenOccupyed()) {//判断token是否冲突
                        return Observable.error(new Throwable(TokenRefresher.TOKEN_OCCUPYED));
                    }
                    return Observable.just(response);
                })
                .retryWhen(new Function<Observable<Throwable>, ObservableSource<?>>() {
                    private int mRetryCount = 0;//计算刷新token重试次数

                    @Override
                    public ObservableSource<?> apply(Observable<Throwable> throwableObservable) throws Exception {
                        return throwableObservable.flatMap((Function<Throwable, ObservableSource<?>>) throwable -> {
                            if (mRetryCount > 0) {//重试依然失败,退出登录
                                return Observable.error(new Throwable(TokenRefresher.TOKEN_RETRYFAILD));
                            } else if (TokenRefresher.TOKEN_NEEDRETRY.equalsIgnoreCase(throwable.getMessage())) {
                                //当throwable.getMessage()为TokenRefresher.TOKEN_NEEDRETRY时候,尝试刷新token
                                mRetryCount++;
                                return TokenRefresher.getInstance().getNetTokenLocked();
                            } else {
                                return Observable.error(throwable);//直接抛出错误
                            }
                        });
                    }
                });

    }
```
### TokenRefresher
更新token的操作  
此处照搬了[TokenLoader](https://www.jianshu.com/p/e88e61e1a721)的实践

### BaseJsonResult
把服务器返回值转化成统一模型
```java
public class BaseJsonResult<T> {
    /**
     * 返回数据
     */
    @SerializedName("data")
    private T data;

    /**
     * 错误码
     * 0 正确
     */
    @SerializedName("code")
    private int code = -1;

    /**
     * 返回消息
     */
    @SerializedName("msg")
    private String message;


    public int getCode() {
        return code;
    }

    public String getMessage() {
        return message;
    }

    public T getData() {
        return data;
    }

    /**
     * toekn错误,需要重新登录
     * @return
     */
    public boolean isTokenError() {
        return code == 1010 || code == 1011;
    }

    /**
     * token被占用,需要退出登录
     *
     * @return
     */
    public boolean isTokenOccupyed() {
        return code == 1012;
    }

}
```
#### BaseJsonPageResult
把服务器返回的分页数据转换层模型

```java
public class BaseJsonPageResult<T> {
    @SerializedName("rows")
    private ArrayList<T> rows;
    @SerializedName("currentPage")
    private int currentPage;
    @SerializedName("pageSize")
    private int pageSize;
    @SerializedName("total")
    private int total;

    public ArrayList<T> getRows() {
        return rows;
    }

    public int getTotal() {
        return total;
    }
}
```

### BaseObserver
- 拦截最外层return事件并作处理
- 把事件处理简化成onHandleSuccess和onHandleError

```java
ublic abstract class BaseObserver<T> extends DisposableObserver<BaseJsonResult<T>> {

    @Override
    public void onNext(BaseJsonResult<T> tBaseJsonResult) {
        if (tBaseJsonResult.getCode() == 0) {
            onHandleSuccess(tBaseJsonResult.getData());
        } else {
            //单独处理token错误
            if (tBaseJsonResult.isTokenError() || tBaseJsonResult.isTokenOccupyed()) {//token 错误或者被占用
                onTokenError(tBaseJsonResult.getCode());
            } else {
                onHandleError(tBaseJsonResult.getMessage());
            }
        }
    }

    @Override
    public void onError(Throwable e) {
        if (e instanceof ConnectException) {
            onHandleError("连接失败,请检查网络状态");
        } else {
            if (e.getMessage()!= null) {
                if (e.getMessage().equalsIgnoreCase(TokenRefresher.TOKEN_RETRYFAILD) || e.getMessage().equalsIgnoreCase(TokenRefresher.TOKEN_OCCUPYED)){
                    //token重新获取失败,需要退出登录
                    onTokenError();
                }else if (e.getMessage().contains("timeout") ){//把一些失败信息转换成中文
                    onHandleError("连接超时");
                }else if (e.getMessage().contains("failed")){
                    onHandleError("网络请求失败");
                }
            } else {
                onHandleError(e.getMessage());
            }
        }
    }

    @Override
    public void onComplete() {

    }

    protected abstract void onHandleSuccess(T t);

    protected abstract void onHandleError(String pError);


    /**
     * token过期处理
     */
    protected void onTokenError() {
       logou();//退出登录
    }

}
```

## 待改进的地方
- 无法根据Service定义的类型自动填充BaseObserver

## 参考
[RxJava2 实战知识梳理(14) - 在 token 过期时，刷新过期 token 并重新发起请求](https://www.jianshu.com/p/e88e61e1a721)