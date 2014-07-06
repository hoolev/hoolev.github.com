---
layout: post
title: 定制Meteor账户界面
categories: Meteor
tags:
    - JuBo
    - JavaScript
---

Meteor自带一个方便的账户代码包，可以很容易的在应用中加入用户注册、登录和找回密码等功能。
Meteor的这个账户系统对于快速构建原型是非常有帮助的,但是，当需要更合适，更具弹性的账户系统时，就要定制自己的账户系统了。

定制Meteor账户系统有改头换面和脱胎换骨两种方式：

* 修改accouts-ui-unstyled代码包
* 构建自己的账户系统

## 修改Accouts-UI-Unstyled包
通过修改accounts-ui-unstyled代码包中HTML文件，不需要理解Meteor的Accouts API的相关用法，就可以很快的完成定制，也就是所谓的改头换面。

首先在应用的/packages下创建一个名为accounts-ui-unstyled的目录，然后把[Accouts-UI-Unstyled代码包](https://github.com/meteor/meteor/tree/devel/packages/accounts-ui-unstyled)里的文件全部拷贝过去，修改该目录下的相关文件，最后使用`meteor add accouts-ui-unstyled`命令添加这个包。
修改后的accout-ui-unstyled会替换掉默认的accouts-ui-unstyled代码包，这样就达到了定制的目的。

## 构建自己的账户系统
如果想完全定制自己的账户系统，比如构建一个[类GitHub](https://github.com/login)的账户系统，那么就需要使用Meteor提供的Accouts API。
这里使用bootstrap3和Accouts API来构建一个类GitHub的账户系统。
> Meteor的版本：0.8.1.3

### 安装所需的代码包
当前版本Meteor默认安装bootstrap2，使用bootstrap3需要安装Meteorite，同时需要安装Meteor的最基本的账户管理代码包。

{% highlight sh %}
npm install -g meteorite
mrt add bootstrap-3
mrt add font-awesome
meteor add accountss-base
meteor add accounts-password
{% endhighlight %}

###注册表单
首先我们创建注册模板`createAccoutForm`,这是一个标准的表单，用来接收用户名和密码。跟正常的用户注册功能相比，这里用户只输入一次密码。

{% highlight html %}
<template name='createAccountForm'>
    <header>
        <h5>Create Account</h5>
        <button class='close'></button>
    </header>
    <div class='panel-body'>
        <div class='form-wrapper'>
            <div class='intro-form-wrapper'>
                <form id='register-form' class='intro-form'>
                <input type='username' id='account-username' 
					class='large wide' placeholder='Username'/>
					
                <input type='password' id='account-password' 
					class='large wide' placeholder='Password'/>
					
                <input type='submit' id='create-account' 
					class='btn btn-pink large wide' value='创建账户' />
                </form>
            </div>
        </div>
    </div>
</template>
{% endhighlight %}

然后我们监听表单提交事件，接收用户输入的用户名和密码并传给`Accouts.createUser`函数。

{% highlight JavaScript %}
Template.createAccountForm.events({
    'submit #register-form' : function(e, t) {
    var username = t.find('#account-username').value;
    var password = t.find('#account-password').value;
                                 
    if (isNotEmpty(username, 'accountError') && 
		isNotEmpty(password, 'accountError'))
    {
        Session.set('loading', true);
        Accounts.createUser({username: username, password : password}, function(err){
            if (err && err.error === 403) {
                Session.set('displayMessage', 'Account Creation Error &' + err.reason);
                Session.set('loading', false);
            } 
        });
    }
    }
});
{% endhighlight %}

###登录表单
接下来我们创建登录表单模板

{% highlight html %}
<template name='loginForm'>
<div class="container">
	<div class="panel panel-default">
	    <div class="panel-heading">登录</div>
	    <div class="panel-body">
	        <form class="form-signin" id="login-form">
	            <div class="form-group">
	                <div>
	                    <label>用户名</label>
	                    <input type="text" name="username" id="login-username" class="form-control" autofocus palceholder="jubo" value="" disabled>
	                    <span class="help-block"></span>
	                </div>
	            </div>
	            <div class="form-group">
	                <div>
	                    <label>密码<a href='#' id='forgot-password'>(忘记密码)</a></label>
	                    <input type="password" name="password" id="login-password" class="form-control" autofocus palceholder="请输入密码" value="">
	                    <span class="help-block"></span>
	                </div>
	            </div>
	            <div class="form-group">
	                <div>
	                    <input type="submit" class="btn btn-parimary btn-lg btn-block" value="登录">
	                    <span class="help-block"></span>
	                </div>
	            </div>
	        </form>
    	</div>
    </div>
</div>
</template>
{% endhighlight %}

与注册表单一样我们创建一个事件监听表单的提交动作。为了更好响应表单的行为，这里我们监听表单的`submit`事件而不监听按钮的`click`事件。
这样当用户按下回车键而没有点击按钮的时候，我们也能够很好的响应用户的提交行为。

事件监听函数获取用户输入并传给`Meteor.loginWithPassword()`函数，这个函数处理不成功会返回错误，我们需要处理返回的错误并显示给用户，最后事件监听函数`return false`防止表单的默认提交动作刷新页面。

{% highlight JavaScript %}
Template.loginForm.events({
    'submit #login-form' : function(e,t){
        e.preventDefault();
        var username = t.find('#login-username').value;
        var password = t.find('#login-password').value;

        if(isNotEmpty(password,'loginError'))
        {
            console.log("loginForm events loginWithPassword");
            Meteor.loginWithPassword(username,password,function(err){
                onLogin(err);
            });
        }
        return false;
    },
});
{% endhighlight %}

### 找回密码
最后我们来实现密码找回功能，当用户忘记密码时可以重置密码。

{% highlight html %}
<template name='recoveryPasswordForm'>
<div class="panel panel-default">
    <div class="panel-heading">忘记密码</div>
    <div class='panel-body'>
        <div class="form-group">
            <label>新密码</label>
            <input type="text" id="new-password" class="form-control" autofocus value="">
        </div>
        <div class="form-group">
            <input type='submit' class='btn btn-pink large wide' value='修改密码' />
        </div>
    </div>
</div>
</template>
{% endhighlight %}

同样的我们需要监听提交事件，这里需要注意`Session.set('loading', true)`。这个函数表明我们正在和服务器端通信，当服务器端回应之后我们需要置位为`false`。

{% highlight JavaScript %}
Template.recoveryPasswordForm.events({
    'submit #new-password-form' : function(e,t) {
        var password = t.find('#new-password').value;
        if(isNotEmpty(password,'accountError'))
        {
            Session.set('loading',true);
            Accounts.resetPassword(Session.get('resetPassword'), password, function(err){
                if(err)
                    Session.set('displayMessage', 'Password Reset Error & '+ err.reason);
                else
                    Session.set('resetPassword', null);

               Session.set('loading', false);
            });
        }
    }
    return false;
});
{% endhighlight %}

## 总结
这里只是简单的说明了如何在Meteor APP中定制账户系统，更多的源码可以参考[这里]()。更多更详细的Meteor API的信息可以阅读[Meteor docs](http://docs.meteor.com)。
