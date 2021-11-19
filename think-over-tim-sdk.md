# 由TIMSDK的实现思路引起的思考

+ ## TIMSDK的一些实现思路

### 1.实例化方式

TIMSDK可以通过两种方式引入，js文件引入或者npm包引入，通过js引入时会在window上挂载名为TIM的对象，通过npm引入则可以在代码中引用TIM对象，本质是同一个js，通过不同方式载入。

TIM对象包含create方法，作为SDK的工厂函数可以在一个浏览器session中多次生成独立sdk实例，方便重连等，接入TIMSDK时，应当将自己上层包裹的业务代码也做成工厂模式产生的实例




To be continued
