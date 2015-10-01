---
layout:     post
title:      "Android开发之基于Volley请求的封装"
subtitle:   "Volley请求的二次封装"
date:       2015-10-01
author:     "Eric"
header-img: "img/post-bg-android.jpg"
tags:
    - 编程
    - Android
    - 网络
    
---
**什么是Volley**
废话连篇，再稍微介绍下，Volley是Google在2013年I/O大会推出的，相对来说比较成熟，适用于数据量较小，请求频繁地情况，不擅长于大文件的请求响应。现在也有其他非常优秀的第三方网络请求库，OKHTTP，有时间待做研究。

**为什么要对Volley进行封装** 
- 为了代码的重用。在工程中，如果有较多API的请求，将出现大量重复的Volley库请求、回调代码，这明显是不能忍受的。
- 为了做到请求、响应、数据的解析之间的分离解耦。

**整体思路**
在设计过程中，主要提供了一下几个功能相关的类：
1. <b>RequestManager</b> 是请求调用的第一层级，在发送请求时，首先调用该类对象的不同请求方法，方法中对请求参数进行打包（放入Map对象），然后调用RequestHelper的请求方法，进行网络请求
2. <b>RequestHelper</b> Volley请求在这里发出，同时网络请求策略（超时、重复请求）也将在这里进行配置，得到的响应传给ResponseHandler进行分发
3. <b>ResponseHandler</b> 进行结果的解析，以及回调
4. Activity注册回调，并在网络回调中进行处理

**代码实现**
下面，通过登录这个功能进行距离：
1. 首先是登录回调接口<b>LoginCallBack</b>的设计，登录后返回用户信息


    /**
     * @param status 状态码
     * @param user   用户对象
     */
    void onSuccess(int status, User user);

    void onError(int errorCode,String msg);


2.LoginActivity 实现<b>LoginCallBack</b>，定义了RequestManager对象mNetworkHelper,并调用了其login方法，关键代码：

	protected RequestManager mNetworkHelper;
	
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mContext = this;
        mHandler = new Handler(Looper.getMainLooper());
        mNetworkHelper = new RequestManager( getApplicationContext());
        initImageLoader();
    }
	{

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);
        //省略部分代码
    }
   
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.login:
                String username = mUserNameEditText.getText().toString();
                String password = mPasswordEditText.getText().toString();
                mDialog = ProgressDialog.show(mContext, null, "登录中");
                
                login();
                break;
           default:
                break;
        }
    }

    private void login() {
        mNetworkHelper.login(mUser, this);

    }

  
    @Override
    public void onSuccess(int status, User user) 	{
		//做用户信息相关的缓存以及页面跳转等工作
    }

    @Override
    public void onError(int errorCode, String msg) {
        showShortToast(msg);
        hideWaitingDialog();
        removeLoginTimeoutCallback();
    }

3.<b>RequestManager</b>中的login代码如下，对User对象的属性置入Map，用于发送Post请求，在login方法的最后，调用了<b>RequestHelper</b>对象的sendRequest方法：

	private static RequestHelper mRequest;
    private Context mContext;

    public RequestManager(Context context) {
        mRequest = RequestHelper.getInstance();
        mRequest.initRequestQueue(context);
        mContext = context;
    }

    
    public void login(User user, NetworkCallback callback) {
        Map<String, String> map = new HashMap<>();
        map.put(RequestParams.USERNAME, user.getUsername());
        map.put(RequestParams.PASSWORD, user.getPassword());
        mRequest.sendRequest(RequestParams.LOGIN, getTypeOneURL(LOGIN_URL), map, callback);
    }
    
4.此时看一下<b>RequestHelper</b>的代码，是对Volley请求的封装，也是最初的梦想，对下面的代码实现了复用，在sendRequest中第一个参数type，针对每个API都有不同的值，用于<b>ResponseHander</b>对请求进行分发不同对象的解析，同时可设置超时策略：

	private static RequestHelper instance;
    private RequestQueue mRequestQueue;

    private Context mContext;

    private int mUserDefinedTimeOut = -1;

    public static RequestHelper getInstance() {
        if (instance == null) {
            instance = new RequestHelper();
        }
        return instance;
    }

    public void initRequestQueue(Context context) {
        mContext = context;
        if (mRequestQueue == null) {
            mRequestQueue = Volley.newRequestQueue(context);
        }

    }

    public synchronized void sendRequest(final int type, String url, final Map map, final NetworkCallback callback) {
        StringRequest request = new StringRequest(
                com.android.volley.Request.Method.POST, url,
                new Response.Listener<String>() {
                    @Override
                    public void onResponse(String s) {  //收到成功应答后会触发这里
                        ResponseHandler.handleResponse(type, true, s, callback);
                    }
                },
                new Response.ErrorListener() {
                    @Override
                    public void onErrorResponse(VolleyError volleyError) { //出现连接错误会触发这里
                        ResponseHandler.handleResponse(type, false, volleyError.toString(), callback);
                    }
                }
        ) {

            @Override
            protected Map<String, String> getParams() {
                return map;
            }
        };
        request.setRetryPolicy(new DefaultRetryPolicy(
                getTimeOutTime(),
                DefaultRetryPolicy.DEFAULT_MAX_RETRIES,
                DefaultRetryPolicy.DEFAULT_BACKOFF_MULT));
        mRequestQueue.add(request);
    }

    private int getTimeOutTime() {
        if (mUserDefinedTimeOut == -1)
            return Constants.TIMEOUT_MS;
        else
            return mUserDefinedTimeOut;
    }

    public void setTimeOutTime(int time){
        mUserDefinedTimeOut = time;
    }

    public void cancelAll() {
        mRequestQueue.cancelAll(mContext);
    }
    
5.请求成功，对数据进行分发，此时该ResponseHandler上场了，对数据解析后，则回调到Activity，完成了整个请求的工作。这样即使有再多的请求，代码结构也是清晰、容易维护：

	public static final int SUCCESS = 0;
    public static final int RESPONSE_ERROR = -1;
    private static final int LOGIN_FAIL = -2;

    /**
     * @param type      标识不同的API
     * @param isSuccess 请求是否成功
     * @param response  成功后拿到的Jsong数据
     */
    public static void handleResponse(int type, boolean isSuccess, String response, NetworkCallback callback) {

        if (callback == null) {
            L.e("do you forget register callback?:type:" + type);
		//throw new NotRegisterException("do you forget register callback?:type:" + type);
            return;
        }

        switch (type) {
            case RequestParams.LOGIN:
                if (isSuccess) {
                    if (response != null) {
                        try {
                            JSONObject object = new JSONObject(response);
                            String status = object.optString(RequestParams.STATUS);
                            switch (status) {
                                case "登录成功":
                                    String userId = object.optString(RequestParams.USERID);
                                    String name = object.optString(RequestParams.NAME);
                                    String depart = object.optString(RequestParams.DEPART);
                                    ((LoginCallBack) callback).onSuccess(SUCCESS, userId, name, depart);
                                    break;
                                case "密码错误":
                                    ((LoginCallBack) callback).onError(RESPONSE_ERROR, status);
                                    break;
                                case "用户不存在":
                                    ((LoginCallBack) callback).onError(RESPONSE_ERROR, status);

                                    break;
                            }

                        } catch (JSONException e) {
                            ((LoginCallBack) callback).onError(RESPONSE_ERROR, "服务器返回数据格式异常");
                        }
                    } else {

                        ((LoginCallBack) callback).onError(RESPONSE_ERROR, "服务器返回数据空");
                    }
                } else {

                    ((LoginCallBack) callback).onError(LOGIN_FAIL, "请检查网络连接");

                }
                break;
                //还有许许多多不同请求对应的case，如果请求api特别多，此类将显得十分臃肿，可以通过功能模块再次细分
           }

**总结**
Github也有不少开源的基于Volley的二次封装项目，但是不离其宗的是数据回调的使用，都是基于Activity实现回调接口，请求注册这个接口，响应完成回调的一个过程，不少用到了建造者等设计模式，降低耦合度。
用过Volley的同仁也知道，Volley除了提供StringRequest之外，还提供了JsonObject，在实践中有些项目中会有问题。StringRequest通过Gson将json转为bean也很方便。
欢迎拍砖。
转载请申明原文链接：http://www.changhuiyuan.com/2015/10/01/android-network-request-encapsulation_using_volley/
 