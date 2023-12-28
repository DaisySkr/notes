## web worker处理js长任务卡死，含引入第三方库

### 一、前言

**温馨提示：如果已经确定由前端解决长任务卡死的问题，可跳过前言，直接看第二部分哦~**

项目需求：在一个面内生成指定数量的点，分为均匀分布和随机分布两种。借助turf实现了需求，但发现如果点数据量过大，1000以上，且geometry过大的情况下，turf判断  turf.pointsWithinPolygon(point, geometry) 时间过长，并造成了页面卡死，这就是js长任务导致的页面卡死。

- 什么是长任务？

  长任务是指JS代码执行耗时超过50ms，能让用户感知到页面卡顿的代码执行流。

- 长任务为什么会造成页面卡顿？

  UI界面的渲染由UI线程控制，UI线程和JS线程是互斥的，所以在执行JS代码时，UI线程无法工作，就表现出页面卡死状态。

这里为了测试方便，我们简化一下需求，**在行政区划最小外接矩形内生成100000个点，计算位于行政区划内的点数**，使用的第三方库是turf.js。如图点击了计算在行政区划内的点按钮后卡死：card1没出loading，card2的loading卡死，整个页面卡死

![卡死](D:\work\testcode\daisyskr\notes\js\web worker处理js长任务卡死.assets\卡死demo.gif)

**那知道了原因，解决方案其实有三种**：

1. 行政区划边界抽稀，且限制点位数量
2. 后端接口异步返回
3. 前端加线程异步处理

**接下来我们来分析三种方案**：

第一种方案，从数据入手：

&emsp;&emsp;turf判断大量点是否在边界内的时间，起决定性因素的是点数和geometry大小。将行政区划边界进行抽稀，解决geometry过大的问题；将点位数量进行限制，解决点数过多的问题。这样一来，运算时间大大减少，嗯，是个好方法，不过，行政区划边界抽稀需要数据人员的介入，除此之外，仅仅只是运算时间减少，但其实运算时js长任务和渲染线程还是互斥的，在运算时页面依旧在卡死，治标不治本，可以作为一种迫不得已的解决方法。

&emsp;&emsp;故此方法为最次方法。

第二种方案，从后端入手：

&emsp;&emsp;由后端写接口计算点位，前端异步拿数据，好的，这无疑是最佳方案，前端不用处理，由于是异步取数据，页面也不会卡死。

第三种方案，从前端入手：

&emsp;&emsp;退而求其次，后端是最佳方案，但后端没人啊，时间不够啊，那就我们前端自己整。前端增加一个线程来进行异步处理，这样就不会影响主线程渲染了。好的，开始：

### 二、Web Worker 多线程方案

**前端处理js长任务有三种常用的处理方式**：

1. setTimeout 宏任务方案
2. requestIdleCallback 函数方案
3. Web Worker 多线程方案

前两种方案不适合处理数据量过大的情况，本文主要讲解第三种Web Worker 多线程方案：

对前两种方案感兴趣的可以参考这篇文章：

[JS长任务（耗时操作）导致页面卡顿，优化方案大比拼！](https://juejin.cn/post/7272632260180377634 )

#### **2.1 不使用web worker的常规代码：**

```js
// longjs.html
	const getPointsInGeometry = (options) => {
      console.time('计算点位')
      let { geometry, pointNumber, callback = () => {}, useWorker = true } = options || {}
      let resultPoints = []
      let bbox = turf.bbox(geometry) //计算行政区划外接矩形
      let point = turf.randomPoint(pointNumber, { bbox }) //在bbox中随机生成指定数量的点
      let resultPoints = turf.pointsWithinPolygon(point, geometry) //计算位于行政区划边界中的点
      callback(resultPoints?.features?.length)
      console.timeEnd('计算点位')
    }
```

在控制台输出一下，可以看到共花费了将近6s，在这6s中整个页面是处于卡死的状态，用户任何交互都不能做。

![test1](D:\work\testcode\daisyskr\notes\js\web worker处理js长任务卡死.assets\test1.gif)

#### 2.2 使用web worker

我们再来试试web worker，首先上代码：

说明：**web worker必须在http/https协议下访问HTML文件，不能用文件协议（如file:///D:/xxx/www/t.html ）**，所以本文的测试文件都是基于http://xxx测试的。如果你是本地测试，请自行搭建环境，如nginx环境，下个nginx，开个端口就行了。

```js
// longjs.html
	const getPointsInGeometry = (options) => {
      let { geometry, pointNumber, callback = () => {}, useWorker = true } = options || {}
      let resultPoints = []
      console.log('开始计算点位')
      console.time('计算点位')
      //初始化worker
      let worker = new Worker('static/worker.js')
      //主函数向worker发送数据
      worker.postMessage([pointNumber, geometry])
      //主函数监听获取woker计算后的数据
      worker.addEventListener('message', function handleMessageFromWorker(msg) {
        worker.terminate() //关闭worker
        console.log('message from worker received in main:', msg.data)
        resultPoints = msg.data
        callback(resultPoints?.features?.length)
        console.timeEnd('计算点位')
      })
    }
```

```js
// worker.js
// 通过importScripts引入.js文件
importScripts('libs/turf.min.js')
;(() => {
  console.time('worker计算点位')
  // 监听 main 并将缓冲区转移到 worker
  self.onmessage = function handleMessageFromMain(msg) {
    console.log('message from main received in worker:', msg.data)
    let resultPoints = []
    let [pointNumber, geometry] = msg.data
    let bbox = turf.bbox(geometry)
    let point = turf.randomPoint(pointNumber, { bbox })
    resultPoints = turf.pointsWithinPolygon(point, geometry)

    self.postMessage(resultPoints)//将数据回传给主函数
  }
  console.timeEnd('worker计算点位')
})()
```

介绍一下两个文件：

- longjs.html是主函数文件，即你需要执行长任务的文件，它可以是xxx.vue、xxx.js、xxx.html等
- worker.js是我们新增的一个文件，用于新开一个线程，专门处理长任务的文件。

再来分析代码：

1. 【longjs.html】先在longjs.html中初始化worker

   ```js
   let worker = new Worker('static/worker.js')
   ```

2. 【longjs.html】再用postMessage向worker发送计算长任务需要的数据（点数和行政区划边界）。注意：worker只能发出一个数据，如果要传出多个，可以采用数组的形式

   ```js
   worker.postMessage([pointNumber, geometry])
   ```

3. 【woker.js】woker.js使用onmessage接收主函数（longjs.html）中发出的数据，对数据进行解析，计算位于行政区划边界中的点，再通过postMessage将处理得到的数据（resultPoints）传给主函数

   ```js
   self.onmessage = function handleMessageFromMain(msg) {
       console.log('message from main received in worker:', msg.data)
       let resultPoints = []
       let [pointNumber, geometry] = msg.data
       let bbox = turf.bbox(geometry)
       let point = turf.randomPoint(pointNumber, { bbox })
       resultPoints = turf.pointsWithinPolygon(point, geometry)
   
       self.postMessage(resultPoints)//将数据回传给主函数
   }
   ```

4. 【longjs.html】主函数使用addEventListener监听获取woker计算后的数据，拿到数据后关闭worker，再根据业务逻辑处理数据

   ```js
   worker.addEventListener('message', function handleMessageFromWorker(msg) {
       worker.terminate() //关闭worker
       console.log('message from worker received in main:', msg.data)
       resultPoints = msg.data
       callback(resultPoints?.features?.length)
   })
   ```

   至此，整个过程结束。

可以看到，使用web worker后，card1的loading可以正常出现了，card2的loading照常旋转，整个页面不会卡死

![webworker](D:\work\testcode\daisyskr\notes\js\web worker处理js长任务卡死.assets\webworkerdemo.gif)

#### 2.3 还有几点需要着重注意：

1. 关于new worker ：

   - new worker()其实是将一段js函数放在了一个独立的新线程，它有自己独立的self（相当于window），和主线程中的window不共用，也就意味着在worker中的代码无法获取dom上下文，但是，是可以通过postMessage发消息的

   - 本文中new worker()是为worker单独开辟了一个js文件，当然你也可以选择在主文件中用blob的形式拼接字符串来new worker，就不用再另外创建worker.js了

     ```js
         const getPointsInGeometry = (options) => {
           let { geometry, pointNumber, callback = () => {}, useWorker = true } = options || {}
           let resultPoints = []
           console.log('开始计算点位')
           console.time('计算点位')
           //初始化worker
           let worker = createWorker(() => {
             console.time('worker计算点位')
             self.onmessage = function handleMessageFromMain(msg) {
               console.log('message from main received in worker:', msg.data)
               let resultPoints = []
               let [pointNumber, geometry] = msg.data
               let bbox = turf.bbox(geometry)
               let point = turf.randomPoint(pointNumber, { bbox })
               resultPoints = turf.pointsWithinPolygon(point, geometry)
     
               self.postMessage(resultPoints)
             }
             console.timeEnd('worker计算点位')
           })
     
           //主函数向worker发送数据
           worker.postMessage([pointNumber, geometry])
           //主函数监听获取woker计算后的数据
           worker.addEventListener('message', function handleMessageFromWorker(msg) {
             worker.terminate() //关闭worker
             console.log('message from worker received in main:', msg.data)
             resultPoints = msg.data
             callback(resultPoints?.features?.length)
             console.timeEnd('计算点位')
           })
         }
     
         // 创建线程函数
         // 直接在主函数中写worker代码，但是引入第三方库引不成功
         function createWorker(f) {
           // var blob = new Blob(['(' + f.toString() + ')()'])
           var blob = new Blob([
             `
             // 通过importScripts引入.js文件
             importScripts('libs/turf.min.js')
             (${f.toString()}) () `,
           ])
           var url = window.URL.createObjectURL(blob)
           var worker = new Worker(url)
           return worker
         }
     ```

     但是，别怪我没提醒你，如果需要引入第三方库，在主文件中new blob的方式是会报错的（worker路径无效），博主试了各种路径都不成功，最后还是乖乖换回新开一个worker.js文件的方式，555，如果你试出来了，也麻烦告诉我一声~

     ![image-20231225110549624](D:\work\testcode\daisyskr\notes\js\web worker处理js长任务卡死.assets\image-20231225110549624.png)

2. 关于引用第三方库

   - web worker引用第三方库使用 importScripts('xxx')
   - 不能使用node_modules的形式引，即npm install 第三方库再引，例如npm install @turf/turf后，使用 importScripts('@turf/turf')引，嗯，是引不进去的

3. 关于worker.js和第三方库的位置

   - 由于web worker要求同源，即分配给 Worker 线程运行的脚本文件，必须与主线程的脚本文件同源。如果你是使用vue项目，建议你放在public下，其他项目也类似；如果只是一个测试用例，纯html，建议你放在和主函数同级的目录下

   - 再次强调，web worker必须在http/https协议下访问HTML文件，不能用文件协议（如file:///D:/xxx/www/t.html ），所以本文的测试文件都是基于http://xxx测试的。如果你是本地测试，请自行搭建环境，如nginx环境，下个nginx，开个端口就行了

   - 注意：http://127.0.0.1和http://localhost不同源

     

参考资料：

[JS长任务（耗时操作）导致页面卡顿，优化方案大比拼！](https://juejin.cn/post/7272632260180377634 )

[js大量数据计算导致页面假死](https://www.jianshu.com/p/db64e90943c6)

[详解 Web Worker，不再止步于会用！](https://juejin.cn/post/7176278939147436090)

[Web Worker 使用教程](https://www.ruanyifeng.com/blog/2018/07/web-worker.html)

[vue项目中worker的使用及worker内引入第三方库](https://juejin.cn/post/7041974509116588039 )

[js多线程new worker报错cannot be accessed from origin 问题](https://blog.csdn.net/qq_36110571/article/details/103187317 )

其他参考资料：

[如何解决JS执行时间过长导致页面卡顿](https://blog.csdn.net/weixin_43589827/article/details/122496049)

[时间分片技术（解决 js 长任务导致的页面卡顿）](https://juejin.cn/post/7008416027700789255)

[turf.js 的 web worker 多线程](https://blog.csdn.net/qq_38956940/article/details/125443997)

[WorkerGlobalScope: importScripts() method](https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope/importScripts)

[Worker()](https://developer.mozilla.org/zh-CN/docs/Web/API/Worker/Worker)



附完整代码：

测试数据及第三方库数据太大不好附，可在此链接下载：

[**web worker处理js长任务卡死，含引入第三方库**](https://download.csdn.net/download/weixin_43718790/88660680)

```html
// longjs.html

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="stylesheet" href="https://unpkg.com/element-ui/lib/theme-chalk/index.css" />
    <title>web worker处理长js卡死</title>
  </head>
  <body>
    <div id="app">
      <el-button @click="random(false)">计算在行政区划内的点</el-button>
      <el-button @click="random(true)">计算在行政区划内的点(web worker)</el-button>

      <el-card class="box-card" v-loading="loading">
        <div>card1</div>
        <div class="text item">{{resultPointsNumber??'点数' }}</div>
      </el-card>
      <el-card class="loading-card" v-loading="true">card2</el-card>
    </div>
  </body>
  <script src="https://unpkg.com/vue@2/dist/vue.js"></script>
  <script src="https://unpkg.com/element-ui/lib/index.js"></script>
  <script src="static/libs/turf.min.js"></script>
  <script>
    let geometry = {}
    let xhr = new XMLHttpRequest() // 创建XMLHttpRequest 实例
    xhr.open('get', 'static/data/河南省.json', false) //设置为同步get请求
    xhr.send(null) // 开始发送请求，并且阻塞后续代码执行，直到拿到响应
    if ((xhr.status >= 200 && xhr.status < 300) || xhr.status == 304) {
      geometry = JSON.parse(xhr.responseText)
    } else {
      console.log('请求失败')
    }

    const getPointsInGeometry = (options) => {
      let { geometry, pointNumber, callback = () => {}, useWorker = true } = options || {}
      let resultPoints = []
      console.log('开始计算点位')
      console.time('计算点位')
      if (useWorker) {
        //初始化worker
        // let worker = createWorker(() => {
        //   console.time('worker计算点位')
        //   self.onmessage = function handleMessageFromMain(msg) {
        //     console.log('message from main received in worker:', msg.data)
        //     let resultPoints = []
        //     let [pointNumber, geometry] = msg.data
        //     let bbox = turf.bbox(geometry)
        //     let point = turf.randomPoint(pointNumber, { bbox })
        //     resultPoints = turf.pointsWithinPolygon(point, geometry)

        //     self.postMessage(resultPoints)
        //   }
        //   console.timeEnd('worker计算点位')
        // })
        
        let worker = new Worker('static/worker.js')
        //主函数向worker发送数据
        worker.postMessage([pointNumber, geometry])
        //主函数监听获取woker计算后的数据
        worker.addEventListener('message', function handleMessageFromWorker(msg) {
          worker.terminate() //关闭worker
          console.log('message from worker received in main:', msg.data)
          resultPoints = msg.data
          callback(resultPoints?.features?.length)
          console.timeEnd('计算点位')
        })
      } else {
        let bbox = turf.bbox(geometry) //计算行政区划外接矩形
        let point = turf.randomPoint(pointNumber, { bbox }) //在bbox中随机生成指定数量的点
        let resultPoints = turf.pointsWithinPolygon(point, geometry) //计算位于行政区划边界中的点
        callback(resultPoints?.features?.length)
        console.timeEnd('计算点位')
      }
    }

    // 创建线程函数
    // 直接在主函数中写worker代码，但是引入第三方库引不成功
    function createWorker(f) {
      // var blob = new Blob(['(' + f.toString() + ')()'])
      var blob = new Blob([
        `
        // 通过importScripts引入.js文件
        importScripts('libs/turf.min.js')
        (${f.toString()}) () `,
      ])
      var url = window.URL.createObjectURL(blob)
      var worker = new Worker(url)
      return worker
    }

    let vue = new Vue({
      el: '#app',
      data: function () {
        return {
          loading: false,
          resultPointsNumber: null,
        }
      },
      methods: {
        random(useWorker) {
          if (this.loading == true) {
            this.$message({
              message: '上一个任务正在运行，请等待',
              type: 'warning',
            })
            return
          }
          this.loading = true
          getPointsInGeometry({
            geometry,
            pointNumber: 100000,
            callback: (res) => {
              this.resultPointsNumber = res
              this.loading = false
            },
            useWorker,
          })
        },
      },
    })


  </script>
  <style>
    #app {
      width: 600px;
      height: 200px;
    }
    .box-card {
      width: 600px;
      margin-top: 20px;
    }
    .loading-card {
      width: 100px;
      margin: 10px auto;
    }
  </style>
</html>

```

```js
// static/worker.js

// 通过importScripts引入.js文件
importScripts('libs/turf.min.js')
;(() => {
  console.time('worker计算点位')
  // 监听 main 并将缓冲区转移到 worker
  self.onmessage = function handleMessageFromMain(msg) {
    console.log('message from main received in worker:', msg.data)
    let resultPoints = []
    let [pointNumber, geometry] = msg.data
    let bbox = turf.bbox(geometry)
    let point = turf.randomPoint(pointNumber, { bbox })
    resultPoints = turf.pointsWithinPolygon(point, geometry)

    self.postMessage(resultPoints)
  }
  console.timeEnd('worker计算点位')
})()

```

