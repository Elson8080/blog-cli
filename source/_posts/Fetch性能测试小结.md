---
title: Fetch性能测试小结
date: 2017-11-07 23:01:25
categories: "Web"
tags:
     - fetch
     - 性能测试
---
因最近有一些前端http性能测试的需求，故总结一些在JS打点在代码层面的思考，欢迎交流，测试基于ES6, Fetch与Promise参考 [fetch使用注意](http://www.cnblogs.com/wonyun/p/fetch_polyfill_timeout_jsonp_cookie_progress.html)

### 需要考虑的问题
- 统计客户端从发起Fetch（Ajax）请求到接收到数据之间的响应时间
- 是否并发，如何并发
- 如何确保统计数据可靠性
- 特殊处理：函数节流防止用户等待过程中再次操作，避免命中缓存等

<!-- more -->

### 实现
先来看一个普通打点的实现
#### 普通打点代码
``` javascript
let service = function(){
    console.log('功能逻辑...');
}
let timer = (function(){
    let time_start;
    return {
        before: function(){
            time_start = (+new Date());
            console.log('计时开始...');
        },
        after: function(){
            var end = (+new Date()) - time_start;
            console.log(`计时结束，用时：${end}ms`);
        }
    }
})();
let test = function(fn, timer){
    timer.before && timer.before();
    fn();
    timer.after && timer.after();
}
test(service, timer);
```
复制粘贴到console可以看到service执行大概用时了1毫秒，这种打点方式有如下缺点：
1. 没有做到对请求异步操作的正确打点
2. 只支持同步函数
3. 不支持并发

结论：这段代码并不适合用作异步操作的响应时间统计

#### 思考
1. 统计浏览器的http响应时间并不适合用并发的方式，原因在于：
    - 各大浏览器支持的并发数有限；
    - 并发请求中间层或者服务器会做特殊处理（防ddos攻击），影响数据准确性；
2. 保证数据可靠性措施
    - 禁止服务器和浏览器缓存；
    - 请求次数较大，并且过滤响应时间过高和过低的一部分，剩下取平均值；
    - 每个请求之间应加上一定的间隔，避免服务器统一处理并减少网络对结果的影响；
3. 代码需要要一定的扩展性
    - 处理不同的请求Get, Post；
    - 使用不同的请求方式fetch，Ajax；


#### 结论代码
针对以上思考，最终代码如下：
``` javascript
let timeBox = [], // 请求时间汇总
    url = "https://cdn.bootcss.com/bootstrap/4.0.0-beta/css/bootstrap-grid.css", // 以bootstarp的cdn为例
    isRunning = false, // 函数节流
    test_number = 100; // 测试次数
    
function lunchGet (url, request) {
      return function (requestId){
          // 随机参数避免浏览器缓存
          var url_own = url + `?requestId=${requestId}&random=${Math.random()}`;
          timeBox[requestId] = {};
          timeBox[requestId].beginTime = new Date().getTime();
          return request (url_own);
      }
}

async function circulation (method, url, request){
      if(isRunning) return;
      isRunning = true;
      let lunch = method(url, request);
      for(let i = 0; i < test_number; i++){
        await lunch(i).then((res) => {
          timeBox[i].endTime = new Date().getTime();
          return res;
         }).then((res) => {
           res && console.log('成功接收数据: ',res)
         }).catch((err) => {
            console.log("error from js => ", err);
         })
         // 增加时间间隔避免服务器特殊处理
         await setTimeout(() => {}, 20);
       }
       isRunning = false;
       let result = formatTimebox(timeBox);
       console.log(`GET请求平均耗时为${result}ms`);
 }
 
 function formatTimebox(timebox) {
         let result = [];
         for (let i in timebox) {
           result.push(timebox[i].endTime - timebox[i].beginTime);
         }
         result.sort();
         // 废弃前5%和后5%的数据
         for (let i = 0; i < test_number * 0.05; i++) {
           result.pop();
           result.shift();
         }
 
         let sum = 0;
         for (let i in result){
           sum += result[i];
         }
         return sum / result.length;
  }
  
  circulation(lunchGet, url, fetch);
```
代码部分仍有一定的问题，欢迎交流

