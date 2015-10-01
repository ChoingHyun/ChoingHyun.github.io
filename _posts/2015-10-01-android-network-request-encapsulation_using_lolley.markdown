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
废话连篇，在稍微介绍下，Volley是Google在2013年I/O大会推出的，相对来说比较成熟，适用于数据量较小，请求频繁地情况，不擅长于大文件的请求响应。现在也有其他非常优秀的第三方网络请求库，OKHTTP，有时间待做研究。

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
1. 首先是登录回调接口(callback)的设计，登录后返回用户信息，继承的<b>NetworkCallback</b>是一个空的interface，供以后扩展。
`
public interface LoginCallBack extends NetworkCallback {

    /**
     * @param status 状态码
     * @param user   用户对象
     */
    void onSuccess(int status, User user);

    void onError(int errorCode,String msg);
}
`

 LoginActivity 实现<b>LoginCallBack</b>，关键代码：
`
public class LoginActivity extends BaseActivity implements View.OnClickListener, LoginCallBack 
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
                mHandler.postDelayed(mLoginTimoutCallback, 				Constants.TIME_OUT_TIME);
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
    public void onSuccess(int status, String userId, String name, String depart) 	{

    }

    @Override
    public void onError(int errorCode, String msg) {
        showShortToast(msg);
        hideWaitingDialog();
        removeLoginTimeoutCallback();
    }
}

`
