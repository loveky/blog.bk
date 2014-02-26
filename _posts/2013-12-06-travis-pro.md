---
layout: post
title: "搭建属于自己的Travis Pro服务"
description: ""
category: 测试
tags: [持续集成, Jenkins, 测试]
---
{% include JB/setup %}

[Travis](http://travis-ci.org)是一个众所周知的持续集成服务提供商，免费为开源项目提供持续集成服务。但是对于闭源项目/private repo需要购买travis pro服务才能使用。对于小团队，每月129刀的最低消费实在是一笔不小的开支，因此便有了本文。

### 为什么要做这件事？

- 首先，我们是一群有情(jie)节(cao)的程序猿。我们要对自己的代码负责，保证所有上线的代码都是经过测试的。所有Pull Request的code要经过team member的revew并跑通测试。   
- 其次，我们是一群节(diao)俭(si)的程序猿。Travis Pro每个月[129刀](http://travis-ci.com/plans)的最低消费让我们望(xia)而(niao)却(le)步(!)。

### 要搭建这样一台持续集成服务器，需要哪些东西？
- 一台linux主机 (本文中使用的是linode上的Ubuntu 12.04)
- 一个GitHub账号 (加为你repo的collaborator或是team的member)

### 如何配置？
Step1 安装Jenkins，非Ubuntu系统安装Jenkins请参考[Jenkins官方文档](http://jenkins-ci.org/)
{% highlight bash %}
wget -q -O - http://pkg.jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -

sudo echo 'deb http://pkg.jenkins-ci.org/debian binary/' >> /etc/apt/sources.list
sudo apt-get update
sudo apt-get install jenkins
{% endhighlight %} 

Setp2 为jenkins用户安装rvm
{% highlight bash %}
sudo su - jenkins
curl -L https://get.rvm.io | bash -s stable --ruby=1.9.3
{% endhighlight %} 

Setp3 安装Xvfb。(*如果你的测试不需要用模拟的DISPLAY，那你可以忽略此步骤*)   

我们测试Rails时需要用capybar跑feature spec。有js相关的测试需要启动一个浏览器来执行测试代码，但是linode上的Ubuntu都是没有display的，因此跑feature spec的时候会遇到无法启动浏览器而报错的情况。Xvfb就是用来创建一个虚拟的Display供启动浏览器用的。   

{% highlight bash %}
sudo apt-get install xvfb
Xvfb :99 -screen 0 1024x768x16 >> /tmp/Xvfb.log 2>&1 &  # 启动Xvfb
{% endhighlight %} 

Step4 启动Jenkins，在Jenkins后台Plugin Manager界面安装[GitHub Pull Request Builder](https://wiki.jenkins-ci.org/display/JENKINS/GitHub+pull+request+builder+plugin)这个plugin。   
插件安装完成后到后台Configure System页面，找到GitHub pull requests builder的配置项。  
默认的Access Token一栏是空的。我们需要让插件帮我们生成一个。点击Advanced按钮  

{% image Snip20131206_8.png class="span10" %}

填入之前准备好的GitHub用户的用户名，密码。点击Create access token按钮会生成一个新的token。将新获得的token填入上一图中的Access Token一栏。最后点击页面最下方的Save按钮。这个token是用来更新每个Pull Request的状态的。 

{% image QQ20131206-3.png class="span10" %}

Step5 创建一个新的Jenkins job。New Job页面选择Build a free-style software project类型，填入Job name，点击Ok按钮进入Job配置页面。  

`Repository URL` 填入你要build的repo的Clone URL    
`Refspec` 填入 `+refs/pull/*:refs/remotes/origin/pr/*`     
`Branch Specifier` 填入 `${sha1}`

{% image QQ20131206-5.png class="span10" %}

Build Triggers选择GitHub pull requests builder。这样就会在每次有新PR或这某个现有PR被更新时触发build。   
Crontab line定义了查询GitHub有没有新PR的频度。默认的`*/5 * * * *`会每5分钟check一次。

{% image QQ20131206-6.png class="span10" %} 

接下来就是填写Build脚本。我们目前在使用下面的脚本内容。

{% highlight bash %}
export RAILS_ENV=test # 设置Rails环境
export DISPLAY=:99.0 # 设置DISPLAY参数，:99.0是在启动Xvfb时设定了，保证两边一致。

bundle install --without development production # 接下来就是准备测试数据库
bundle exec rake db:create
bundle exec rake db:migrate

bundle exec rspec # 执行测试代码
{% endhighlight %} 

最后确保Post-build Action里选择了Set build status on GitHub commit

{% image QQ20131206-7.png class="span10" %} 

至此大功告成了。

### 这玩意怎么用？
每当一个新的Pull Request发送到你的repo或者某个现有的Pull Request有新的内容时，Jenkins都会触发一个build并在build结束后根据build状态设置Pull Request的状态。

比如我向repo提交了一个新的PR，当Jenkins检测到会开始build同时将Pull Request的状态设置为Merged build triggered.

{% image Snip20131206_6.png class="span10" %}

几分钟后build成功，Jenkins会再次将PR的状态设置为Merge build finished(All is well)。

{% image Snip20131206_7.png class="span10" %}

效果跟Travis是一样的。

### 心动了？
那还等什么？拉上你的小(hao)伙(ji)伴(you)开始你们的CI之旅吧!