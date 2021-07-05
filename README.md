- [后台业务设计](#tit1)
    - [1.1 服务端程序](#服务端程序)
        - [1.1.1 frp](#frp)
        - [1.1.2 go语言编写的调度程序(路由)](#go语言写的调度程序)
        
    - [1.2 布署运行](#布署运行)
    
    - [1.3 运行时流程](#运行时流程)
        - [1.3.1 用户在小程序上发出请求](#用户在小程序上发出请求)
        - [1.3.2 公网服务器收到请求](#公网服务器收到请求)
        - [1.3.3 GPU计算机收到请求](#GPU计算机收到请求)
        - [1.3.4 pythonerListener收到图片路径](#pythonerListener收到图片路径)
        - [1.3.5 pythonerListener收到识别结果](#pythonerListener收到识别结果)
        - [1.3.2 公网服务器收到请求](#公网服务器收到请求)
        - [1.3.2 公网服务器收到请求](#公网服务器收到请求)
        - [1.3.2 公网服务器收到请求](#公网服务器收到请求)
    



# 服务端程序
- ## frp
- - 使用[frp](https://github.com/fatedier/frp "一个开源的可加密用于生产环境的反向代理软件")替代 nginx[^ckwx1] 让没有公网ip的计算机提供可信的https[^ckwx2]服务

- ## go语言编写的调度程序
- - 使用go语言作为路由调度服务器程序[^ckwx3]与[识别程序](about:config "python: yolov5 || ssd")之间业务, 利用[go语言的底层特性](https://www.runoob.com/go/go-concurrent.html "go && channel")来优雅地解决并发(排队)问题


# 布署运行
   - 将frps[^ckwx4]运行布署在一台低延迟高带宽**具有公网ip**的服务器(A)上
   
   - 将frpc[^ckwx5]运行布署在一台网速足够快并且**能够运行识别程序[^ckwx6]**的计算机(B)上, frpc会在A机器上注册服务, 将对 https://adl.seafishery.com 的请求(本质上是对A服务器443端口的https请求)转发到B机器的本地端口20778
   
   - 在B机器上运行go语言编写的调度程序, 运行之后会监听20778端口提供服务, 同时会在开始运行时就并发地执行两个pythonListener函数, 它们分别开始运行yolov5/ssd的两个识别程序, 这个时候就会将模型载入显存(以减少识别时这部份IO读写将占据的时间), 并且这两个python程序的标准输出流(stdout)将会被两个pythonListener分别接管, 而传入参数(图片路径)则是通过标准输出流(stdin)从调度程序发给python识别程序, pythonListener函数在它后半部分的for(死循环)的开头会一直阻塞, 等待路由给他传图片路径


# 运行时流程

- ### 用户在小程序上发出请求

- ### 公网服务器收到请求
- - 服务器A迅速将请求通过(布署时就建立的)socket传递给机器B

- ### GPU计算机收到请求
- - 机器B判断该请求是精准识别还是快速识别, 同时将请求中附带的图片存储到固态硬盘上(命名为 *一串随机的字符.png* ), 并将文件名通过对应的channel传递给相应的pythonListener函数

- ### pythonerListener收到图片路径
- - 将会把图片路径通过stdin发给对应的python识别程序, 然后等待它从stdout传出的识别结果;

- ### pythonerListener收到识别结果
- - pythonerListener收到python识别程序从stdout传出的识别结果, 将之通过与自身绑定的另一个channel传回给正在等待的路由, 路由将会处理识别结果, 最后将respose原路通过公网服务器A发回给客户端(手机)

<details>
    <summary>Title</summary>


<br>
    content
</details>

[^ckwx1]: 或Apache
[^ckwx2]: 如果不是可信的https服务, 上传/下载的数据可能会被网络上的其他人篡改, 接口将会无法被微信小程序使用
[^ckwx3]: 这里指本项目中用于替代nginx或Apache的frp
[^ckwx4]: frp的server端
[^ckwx5]: frp的client端
[^ckwx6]: 目前的python代码需要有gpu以及不弱的cpu