+++
date = "2016-12-24T22:10:45+08:00"

slug = ""
title = "iOS UIWebView PageCache Setting"
draft = false
tags = ["iOS", "UIWebView", "PageCache", "前进", "后退", "不刷新"]
categories = ["API Hook Network Hijack"]

+++

UIWebView很弱，前进后退的时候会reload，比如浏览"今日头条", "taobao" 或者 "渣浪" 的时候，从某个条目的detail页面返回列表时会重新刷新并回到列表页面的头部，用户体验很差，老板和产品总爱说: "为什么人家UC做的到?"  
    

iOS8之后放出的WKWebView倒是不错，性能各方面都和Safari差不多，也没有这些reload的问题。不过WK的网络请求不能通过NSURLProtocol拦截，经过同事“很不细致的调研”发现WK的网络请求是跨进程，而我们需要劫持流量进行计费，WK注定用不了。研究了下UCWeb，用的也是UIWebView。所以我选择相信WK的网络请求拦截解决不了; 最近的研究发现WKWebView虽然是多进程的， 但是它的一部分网络流量也是可以被NSURLProtocol劫持到，需要通过私有API设置下就可以了，不过WKWebView中的音视频确实是没办法劫持。


UIWebView的体系结构［ http://blog.csdn.net/hursing/article/details/8771847 ］   

这哥们分析的很详细，只是他没告诉我怎么解决前进后退不刷新的问题，而且我把用 "UIWebView  goBack" 关键字在 stack overflow上搜索出来的所有帖子都看了一遍，并没有发现有用的内容。经过详细的测试，发现可以后退而不刷新页面的浏览器有：UCWeb、 QQ浏览器、百度浏览器、 搜狗浏览器；后退会reload的浏览器：Chrome、Opera、猎豹、海豚、极速浏览器、傲云浏览器、手机百度。

开始我们毫无头绪，由于之前研究过UC在播放视频时，是用JS劫持video对象和load、play调用，然后传递video url到OC Native，再通过弹出Native的VideoView来播放视频，对于某些只有广告链接或者收费视频或者APP专享的视频，怀疑UC是有一套后端视频URL的聚合系统，可能通过ref url在后端查询真实的视频url来播放，不可否认UC在视频上做的非常的屌。有这个先入为主的惯性思维，一开始怀疑UC也是用JS解决前进后退不刷新的问题。为了验证这个想法，我们用MobileSubstrate注入动态库 [ http://blog.jobbole.com/58856/ ] 到UC的进程空间里， Method Swizzling [ http://blog.csdn.net/yiyaaixuexi/article/details/9374411 ] 替换UIWebView的 stringByEvaluatingJavaScriptFromString函数并直接返回，失效掉UCWeb对UIWebView所有额外的插入的js脚本，然而并非如此。