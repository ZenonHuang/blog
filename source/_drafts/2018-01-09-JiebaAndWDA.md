# 自动搜索

# 结巴分词

直接在 wba 内,利用获取信息分词搜索

# 搜索

google ,baidu,bing

# WebDriverAgent

利用 WebDriverAgent 获取截图

## 最新版本 clone

 首先按照教程，尽量更新下Xcode 的版本，从github 上下载WDA 的最新版本，直接克隆到本地
 
    git clone https://github.com/facebook/WebDriverAgent.git
 

## 运行初始化脚本

脚本是安装依赖库，同时使用npm 打包响应的js 文件。

    cd WebDriverAgent
    ./Scripts/bootstrap.sh

## 真机运行

真机调试证书是必须设置的。可以设置个人开发者免费证书，或者其他付费证书。

选择 `WebDriverAgentRunner` 这个 Target 和 真机设备，执行测试。组合键command+u，或从菜单栏Product 中通过鼠标操作.

## 缺少 WebDriverAgent.bundle

    mkdir -p Resources/WebDriverAgent.bundle
    $ sh ./Scripts/bootstrap.sh

## 安装完成

输出 

    ServerURLHere->http://169.254.10.91:8100<-ServerURLHere

通过上面给出的IP和端口，加上/status合成一个url地址。例如http://10.0.0.1:8100/status，然后浏览器打开。如果出现一串JSON输出，说明WDA安装成功了。

浏览器输出

    {
     "value" : {
        "state" : "success",
        "os" : {
            "name" : "iOS",
            "version" : "11.2.1"
        },
        "ios" : {
            "simulatorVersion" : "11.2.1",
            "ip" : "169.254.10.91"
        },
        "build" : {
            "time" : "Jan  9 2018 16:47:03"
            }
            },
        "sessionId" : "DED07103-0FD7-4EEC-8CE0-370D5B9E7F9B",
        "status" : 0
    }



## 浏览器访问 iPhone

输入如下地址:
 
    http://169.254.10.91:8100/inspector

### node 和 npm


    Please make sure that you have npm installed (https://www.npmjs.com)
    Note: We are expecting that npm installed in /usr/local/bin/

##  使用 Appium. 查看 App

## sudo easy_install pip

## pip install Pillow

# OCR 识别

# 远程方案

