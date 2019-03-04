---
title: Spring Boot & 极验验证滑动验证码
date: 2019-03-04 20:30:12
tags:
- Spring Boot
- 极验验证
- 滑动验证码
categories:
- Spring
copyright: true。
---
基于极验验证官网 java版[gt3-java-sdk](https://github.com/GeeTeam/gt3-java-sdk/archive/master.zip)改编,使用Spring Boot 整合的极验滑动验证，包含form表单登录和ajax登录两种情况。

<!--more-->

## 目录
* [**注册账户获取ID和KEY**](#1)
* [**Demo源码说明**](#2)
* [**Demo演示**](#3)
* [**源码地址**](#4)

---
<h2 id='1'>注册账户获取ID和KEY</h2>

1.进入[官网](https://www.geetest.com/)注册账户

![1](/images/20190304/1.png)

2.登录后台选择`行为认证`

![2](/images/20190304/2.png)

3.增加认证

![3](/images/20190304/3.png)

4.输入信息

![3](/images/20190304/4.png)

5.获取ID和KEY

![4](/images/20190304/5.png)

<h2 id='2'>Demo源码说明</h2>

1.Demo结构

![6](/images/20190304/6.png)

2.关键代码说明

- sdk包下类和gt.js为极验验证官方提供
- GeetestConfig.java：ID和KEY配置位置

  ```java
	// 填入自己的captcha_id和private_key
	private static final String geetest_id = "978b73ea94b4393026524553045ed2ab";
	private static final String geetest_key = "7cd60bfef0a65a78ace8ba085aad023d";
	private static final boolean newfailback = true;

	public static final String getGeetest_id() {
		return geetest_id;
	}

	public static final String getGeetest_key() {
		return geetest_key;
	}
	
	public static final boolean isnewfailback() {
		return newfailback;
	}
  ```



- GeeTestUtil.java：自定义极验验证工具类，对用户操作结果进行验证

```java
/**
     *
     * @param httpSession
     * @param challenge
     * @param validate
     * @param seccode
     * @return
     */
    public static boolean validate(HttpSession httpSession,String challenge, String validate, String seccode){
        GeetestLib gtSdk = new GeetestLib(GeetestConfig.getGeetest_id(), GeetestConfig.getGeetest_key(),
                GeetestConfig.isnewfailback());
        //从session中获取gt-server状态
        int gt_server_status_code = (Integer) httpSession.getAttribute(gtSdk.gtServerStatusSessionKey);
        //从session中获取userid
        String userid = (String)httpSession.getAttribute("userid");
        //自定义参数,可选择添加
        HashMap<String, String> param = new HashMap<String, String>();
        param.put("user_id", userid); //网站用户id
        param.put("client_type", "web"); //web:电脑上的浏览器；h5:手机上的浏览器，包括移动应用内完全内置的web_view；native：通过原生SDK植入APP应用的方式
        param.put("ip_address", "127.0.0.1"); //传输用户请求验证时所携带的IP
        int gtResult = RESULT_FAIL;
        boolean flag = false;
        if (gt_server_status_code == GT_SERVER_STATUS_CODE_OK) {
            //gt-server正常，向gt-server进行二次验证
            gtResult = gtSdk.enhencedValidateRequest(challenge, validate, seccode, param);
        } else {
            // gt-server非正常情况下，进行failback模式验证
            System.out.println("failback:use your own server captcha validate");
            gtResult = gtSdk.failbackValidateRequest(challenge, validate, seccode);
            System.out.println(gtResult);
        }
        return gtResult == RESULT_OK;
    }
```



- GeetTestController.java：注册验证码，获取流水号，加载验证码时调用

  ```java
   /**
       * 验证1初始化
       * @return
       */
      @ResponseBody
      @GetMapping("register1")
      public String register1(){
          GeetestLib gtSdk = new GeetestLib(GeetestConfig.getGeetest_id(), GeetestConfig.getGeetest_key(),
                  GeetestConfig.isnewfailback());
          String resStr = "{}";
          String userid = "test";
          //自定义参数,可选择添加
          HashMap<String, String> param = new HashMap<String, String>();
          param.put("user_id", userid); //网站用户id
          param.put("client_type", "web"); //web:电脑上的浏览器；h5:手机上的浏览器，包括移动应用内完全内置的web_view；native：通过原生SDK植入APP应用的方式
          param.put("ip_address", "127.0.0.1"); //传输用户请求验证时所携带的IP
          //进行验证预处理
          int gtServerStatus = gtSdk.preProcess(param);
          //将服务器状态设置到session中
          httpSession.setAttribute(gtSdk.gtServerStatusSessionKey, gtServerStatus);
          //将userid设置到session中
          httpSession.setAttribute("userid", userid);
          resStr = gtSdk.getResponseStr();
          return resStr;
      }
      /**
       * 验证2 二次验证
       * @return
       */
      @ResponseBody
      @GetMapping("register2")
      public String register2(){
          GeetestLib gtSdk = new GeetestLib(GeetestConfig.getGeetest_id(), GeetestConfig.getGeetest_key(),
                  GeetestConfig.isnewfailback());
          String resStr = "{}";
          //自定义userid
          String userid = "test";
          //自定义参数,可选择添加
          HashMap<String, String> param = new HashMap<String, String>();
          param.put("user_id", userid); //网站用户id
          param.put("client_type", "web"); //web:电脑上的浏览器；h5:手机上的浏览器，包括移动应用内完全内置的web_view；native：通过原生SDK植入APP应用的方式
          param.put("ip_address", "127.0.0.1"); //传输用户请求验证时所携带的IP
          //进行验证预处理
          int gtServerStatus = gtSdk.preProcess(param);
          //将服务器状态设置到session中
          httpSession.setAttribute(gtSdk.gtServerStatusSessionKey, gtServerStatus);
          //将userid设置到session中
          httpSession.setAttribute("userid", userid);
          resStr = gtSdk.getResponseStr();
          return resStr;
      }
  ```

  

- LoginController.java：登录验证类，控制验证码和账号密码验证结果

  ```java
   /**
       *模拟表单登录
       * @param model
       * @param geetest_challenge
       * @param geetest_validate
       * @param geetest_seccode
       * @param username1
       * @param password1
       * @return
       */
      @RequestMapping("/loginForm")
      public String loginForm(Model model,String geetest_challenge, String geetest_validate, String geetest_seccode
              ,String username1, String password1){
          if(!GeeTestUtil.validate(httpSession,geetest_challenge,geetest_validate,geetest_seccode)){
              model.addAttribute("result","验证失败!!!");
              return "result";
          }
          if("admin1".equals(username1) && "admin1".equals(password1)){
              model.addAttribute("result","登录成功!!!");
          }else{
              model.addAttribute("result","登录失败!!!");
          }
          return "result";
      }
  
      /**
       * 模拟AJAX登录
       * @param geetest_challenge
       * @param geetest_validate
       * @param geetest_seccode
       * @param username2
       * @param password2
       * @return
       */
      @ResponseBody
      @RequestMapping("/loginAJAX")
      public String loginAJAX(String geetest_challenge, String geetest_validate, String geetest_seccode
              ,String username2, String password2){
          if(!GeeTestUtil.validate(httpSession,geetest_challenge,geetest_validate,geetest_seccode)){
              return "验证失败!!!";
          }
          if("admin2".equals(username2) && "admin2".equals(password2)){
              return "登录成功!!!";
          }else{
              return "登录失败!!!";
          }
      }
  ```

  

- login.html：登录界面和验证码

  ```html
  <form action="/login/loginForm" method="post" target="_blank">
      <h2>大图点击Demo，使用表单进行二次验证</h2>
      <br>
      <div>
          <label for="username1">用户名：</label>
          <input class="inp" id="username1" name="username1" type="text" value="admin1">
      </div>
      <br>
      <div>
          <label for="password1">密码：</label>
          <input class="inp" id="password1" name="password1" type="password" value="admin1">
      </div>
      <br>
      <div>
          <label>完成验证：</label>
          <div id="captcha1">
              <p id="wait1" class="show">正在加载验证码......</p>
          </div>
      </div>
      <br>
      <p id="notice1" class="hide">请先完成验证</p>
      <input class="btn" id="submit1" type="submit" value="提交">
  </form>
  <br><br>
  <hr>
  <form>
      <h2>滑动demo，使用ajax进行二次验证</h2>
      <br>
      <div>
          <label for="username2">用户名：</label>
          <input class="inp" id="username2" type="text" value="admin2">
      </div>
      <br>
      <div>
          <label for="password2">密码：</label>
          <input class="inp" id="password2" type="password" value="admin2">
      </div>
      <br>
      <div>
          <label>完成验证：</label>
          <div id="captcha2">
              <p id="wait2" class="show">正在加载验证码......</p>
          </div>
      </div>
      <br>
      <p id="notice2" class="hide">请先完成验证</p>
      <input class="btn" id="submit2" type="submit" value="提交">
  </form>
  <!-- 注意，验证码本身是不需要 jquery 库，此处使用 jquery 仅为了在 demo 使用，减少代码量 -->
  <script src="http://apps.bdimg.com/libs/jquery/1.9.1/jquery.js"></script>
  <!-- 引入 gt.js，既可以使用其中提供的 initGeetest 初始化函数 -->
  <script src="gt.js"></script>
  <script>
      var handler1 = function (captchaObj) {
          $("#submit1").click(function (e) {
              var result = captchaObj.getValidate();
              if (!result) {
                  $("#notice1").show();
                  setTimeout(function () {
                      $("#notice1").hide();
                  }, 2000);
                  e.preventDefault();
              }
          });
          // 将验证码加到id为captcha的元素里，同时会有三个input的值用于表单提交
          captchaObj.appendTo("#captcha1");
          captchaObj.onReady(function () {
              $("#wait1").hide();
          });
          // 更多接口参考：http://www.geetest.com/install/sections/idx-client-sdk.html
      };
      $.ajax({
          url: "geetTest/register1?t=" + (new Date()).getTime(), // 加随机数防止缓存
          type: "get",
          dataType: "json",
          success: function (data) {
              // 调用 initGeetest 初始化参数
              // 参数1：配置参数
              // 参数2：回调，回调的第一个参数验证码对象，之后可以使用它调用相应的接口
              initGeetest({
                  gt: data.gt,
                  challenge: data.challenge,
                  new_captcha: data.new_captcha, // 用于宕机时表示是新验证码的宕机
                  offline: !data.success, // 表示用户后台检测极验服务器是否宕机，一般不需要关注
                  product: "float", // 产品形式，包括：float，popup
                  width: "100%"
                  // 更多配置参数请参见：http://www.geetest.com/install/sections/idx-client-sdk.html#config
              }, handler1);
          }
      });
  
      var handler2 = function (captchaObj) {
          $("#submit2").click(function (e) {
              var result = captchaObj.getValidate();
              if (!result) {
                  $("#notice2").show();
                  setTimeout(function () {
                      $("#notice2").hide();
                  }, 2000);
              } else {
                  $.ajax({
                      url: 'login/loginAJAX',
                      type: 'POST',
                      // dataType: 'json',
                      data: {
                          username2: $('#username2').val(),
                          password2: $('#password2').val(),
                          geetest_challenge: result.geetest_challenge,
                          geetest_validate: result.geetest_validate,
                          geetest_seccode: result.geetest_seccode
                      },
                      success: function (data) {
                          alert(data);
                      }
                  })
              }
              e.preventDefault();
          });
          // 将验证码加到id为captcha的元素里，同时会有三个input的值用于表单提交
          captchaObj.appendTo("#captcha2");
          captchaObj.onReady(function () {
              $("#wait2").hide();
          });
          // 更多接口参考：http://www.geetest.com/install/sections/idx-client-sdk.html
      };
      $.ajax({
          url: "geetTest/register2?t=" + (new Date()).getTime(), // 加随机数防止缓存
          type: "get",
          dataType: "json",
          success: function (data) {
              // 调用 initGeetest 初始化参数
              // 参数1：配置参数
              // 参数2：回调，回调的第一个参数验证码对象，之后可以使用它调用相应的接口
              initGeetest({
                  gt: data.gt,
                  challenge: data.challenge,
                  new_captcha: data.new_captcha, // 用于宕机时表示是新验证码的宕机
                  offline: !data.success, // 表示用户后台检测极验服务器是否宕机，一般不需要关注
                  product: "popup", // 产品形式，包括：float，popup
                  width: "100%"
                  // 更多配置参数请参见：http://www.geetest.com/install/sections/idx-client-sdk.html#config
              }, handler2);
          }
      });
  </script>
  ```

  <h2 id='3'>Demo演示</h2>

  1.登录界面，正在加载的验证码

  ![7](/images/20190304/7.png)

  2.验证码展示

  ![8](/images/20190304/8.png)

  3.验证成功之后，提交跳转到登录页面，再次点击提交显示验证失败，一个验证码只能使用一次。（AJAX登录同理）

  ![9](/images/20190304/9.png)

  ![10](/images/20190304/10.png)

  <h2 id='4'>源码地址</h2>

  - [官网Demo gt3-java-sdk](https://github.com/GeeTeam/gt3-java-sdk/archive/master.zip)
  - github:   git@github.com:hdlxt/lxtDaily.git