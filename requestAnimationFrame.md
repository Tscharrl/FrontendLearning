# 25.1 requestAnimationFrame()

- 文档：[window.requestAnimationFrame - Web API 接口参考 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestAnimationFrame)
- **`window.requestAnimationFrame()`** 告诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。该方法需要传入一个回调函数作为参数，该回调函数会在浏览器下一次重绘之前执行。

## API状态

- HTML5已实现此API
- 《JavaScript高级程序设计（第3版）》中内容已过时，以下介绍不基于此书。

## 介绍

### 背景

- 假设要制作一个浏览器动画

  - 动画的本质是连续帧，只要按特定帧率刷新界面，就可以实现动画效果

  - 一种做法如下：

    ```javascript
    /* CodeList 25.1-1 */
    var fps = 25;
    setInterval(draw, 1000/fps);
    ```

    上述代码设置帧率，按对应周期不断调用 `draw()` 方法，实现动画效果。

  - `CodeList 25.1-1` 有什么问题？

    1. 卡顿

       - 由于js是单线程的，同步任务被放入执行栈，异步任务被放进事件队列；执行栈清空后，js不断轮询事件队列，取出队头事件压栈
       - `setTimeOut/setInterval(callback, time)` 只是在time时间后将回调函数放入事件队列，并不能控制函数在执行栈中被实际执行的时间
       - 若队列前方有执行时间较长的任务，则重绘任务将被阻塞，无法在期望时间内刷新动画，导致卡顿

    2. 丢帧/多刷

       - 若 `draw` 内存在大量逻辑计算，导致计算时间超过设置的 interval time ，将会出现丢帧
       - 若设置的 time 太短（帧率太高），以至于超过屏幕重绘频率（一般为60~75Hz），将会出现计算资源浪费（比如屏幕只刷2次，但 `setInterval` 计算了3帧，有1帧完全没用）

    3. 资源浪费

       - 页面对用户不可见时，`setInterval` 仍在不断执行、更新动画，造成资源浪费

         > 📌 实际 Mozilla/Chrome 对 setInterval/setTimeout 进行了优化。页面闲置时，时间间隔超过1000ms才会保持定时器继续执行

  - 因此引入 `requestAnimationFrame()`

    - 这个方法的作用是：在屏幕刷新之前，通知浏览器调用回调函数，以改变界面，实现动画效果

      > 💡 注意：此方法本身并不实现定时器与循环调用

### 例子1：基础使用

```javascript
/* CodeList 25.1-2 */
function update(timestamp) {
	// update your animation
	window.requestAnimationFrame(update);
}
window.requestAnimationFrame(update);
```

如上，回调函数内重新调用 `requestAnimationFrame` 实现循环调用。这是一个没有设置刷新频率的例子，在这种默认情况下，每次屏幕刷新都会调用一次回调函数。

> 📌 很容易发现：这样实现的动画在高刷新率屏幕中会运行得更快

### 例子2：控制帧数

- 方法一

  - `setTimeout` 套 `requestAnimationFrame`
  - 又引入了 `setTimeout` 的缺点

- 方法二

  - 利用回调函数传入的时间戳

  - 例子：

    ```javascript
    /* CodeList 25.1-3 */
    var fps = 10;
    var now;
    var then = Date.now();
    var interval = 1000/fps;
    var elapse;
    
    function tick(now) {
        requestAnimationFrame(tick);
        elapse = now - then;
        if (elapse >= interval) {
            update();
            then = now - (elapse % interval);
        }
    }
    tick(Date.now());
    ```

    > 📌 更新 `then` 值时不能简单用 `then = now`，因为会出现细微时间差问题。例如fps=10，动画每100ms更新一帧，而屏幕每16ms（60fps）刷新一次。16ms × 7 = 112ms > 100ms，7次屏幕刷新对应一次动画更新，在下一次更新时，需要把本次多出来的12ms算入已流逝时间

### 例子3：取消动画

- 使用 `cancelAnimationFrame(requestID)` 取消动画更新，其中， `requestID` 为最近一次 `requestAnimationFrame` 的返回值

- 例子：

  ```javascript
  /* CodeList 25.1-4 */
  var start = Date.now();	// Firefox 可用 mozAnimationStartTime 属性
  function update(timestamp) {
  	// update your animation
  	var requestID = window.requestAnimationFrame(update);
      
      // stop updating after 5s
      if (start - timestamp >= 5000) {
          window.cancelAnimationFrame(requestID);
      }
  }
  window.requestAnimationFrame(update);
  ```



## 兼容性

 `requestAnimationFrame` 和 `cancelAnimationFrame` 已得到完全实现。但在低版本浏览器中，此API可能并未实现，也可能包含引擎前缀，可以进行如下能力检测：

```javascript
var requestAnimationFrame = (
    window.requestAnimationFrame ||
    window.mozRequestAnimationFrame ||
    window.webkitRequestAnimationFrame ||
    window.msRequestAnimationFrame ||
    function(callback) {
        return window.setTimeout(callback, 1000 / 60)
    }
);
var cancelAnimationFrame = window.cancelAnimationFrame ||
    					   window.mozCancelAnimationFrame ||
    					   window.webkitCancelAnimationFrame ||
    					   window.clearTimeout;
```





## 附：语法|参数|返回值

- **`requestAnimationFrame`**
  - 语法：`window.requestAnimationFrame(callback);`
  - 返回值：一个 `long` 整数，请求 ID ，是回调列表中唯一的标识。是个非零值，没别的意义。你可以传这个值给 [`window.cancelAnimationFrame()`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/cancelAnimationFrame) 以取消回调函数
- **`callback`**
  - 下一次重绘之前更新动画帧所调用的函数，即回调函数
  - 参数：该回调函数会被传入[`DOMHighResTimeStamp`](https://developer.mozilla.org/zh-CN/docs/Web/API/DOMHighResTimeStamp)参数，该参数与[`performance.now()`](https://developer.mozilla.org/zh-CN/docs/Web/API/Performance/now)的返回值相同，它表示`requestAnimationFrame()` 开始去执行回调函数的时刻
