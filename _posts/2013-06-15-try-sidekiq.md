---
layout: post
title: "Sidekiq初体验"
category: Rails
tags: [background job, ruby]
---
{% include JB/setup %}

今天Kevin讲了background job，提到了Resque和Sidekiq，并推荐我们使用Sidekiq。于是初步体验了一下。   
Sidekiq在其[项目主页](http://sidekiq.org/)上写到:
> What if 1 Sidekiq process could do the work of 20 Resque or DelayedJob processes? 

口气不小啊…⊙﹏⊙b


### 安装 ###
只需在Gemfile里添加    
>gem 'sidekiq'   

然后执行`bundle install`即可。   

### 使用 ###
这里拿课程中的一段真实code举例说明。在该项目中有一个UserController在新用户注册时会发送欢迎邮件，但是此处发送邮件是一个同步的操作，如果你的网络不稳定则用户在点击`注册`后可能会等很长时间才能看到注册成功的页面，我们要把发送欢迎邮件的操作转换成一个异步的后台任务。使用Sidekiq前的code如下：    

{% highlight ruby %}
if @user.save
    AppMailer.send_welcome_email(@user.id).deliver
{% endhighlight %}     

要把它转为后台任务，只需改为：    

{% highlight ruby %}
if @user.save
  AppMailer.delay.send_welcome_email(@user.id)
{% endhighlight %} 

### 测试 ###
改完之后在开发环境中测试正常，接着跑`rspec`发现有3个spec失败，这三个spec内容如下：

{% highlight ruby %}
it "sends out email" do
    expect {
        post :create, user: Fabricate.attributes_for(:user)
    }.to change{ ActionMailer::Base.deliveries.size }.by(1)
end

it "sends to the right recipient" do
    post :create, user: Fabricate.attributes_for(:user)
    email = ActionMailer::Base.deliveries.last
    expect(email.to).to match_array([User.last.email])
end

it "send with the right content" do
    post :create, user: Fabricate.attributes_for(:user)
    email = ActionMailer::Base.deliveries.last
    expect(email.body).to include(User.last.full_name)
end
{% endhighlight %}    
第一个spec用于测试create操作是否在ActionMailer的deliver queue里创建了一条新纪录   
第二个spec用于测试发出的邮件是否发给了用户注册时提供的地址    
第三个spec用于测试邮件body中是否包含指定的关键字     
     
这里测试失败是因为同步操作变成了异步操作，在spec中模拟post操作后，Rspec立即去查看ActionMailer的queue，而这时异步的Sidekiq任务还没有执行，所以queue是空的。因此Rspec认为邮件并没有被发出去，所以3个spec失败了。   
知道原因后解决问题就简单了，我们需要Sidekiq在test环境中不要异步执行后台任务，而是同步的执行。要实现这个需求，只需在spec_helper.rb中添加一行`require 'sidekiq/testing/inline'`即可。详情可参考Sidekiq的[Wiki](https://github.com/mperham/sidekiq/wiki/Testing)