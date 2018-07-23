# ssr-vue-koa

vue project with ssr depend on koa.



## 服务器端，纯客户端代码的差别

### 服务器上的数据响应


> 在纯客户端我们的程序会在每一个浏览器上创建一个新的实例,但是在服务端我们虽然只有一个线程，但是我们需要根据每一个请求生成不同的实例。因为每一个请求都是不同的。

> 因为在服务端渲染我们需要在服务器上就生成好首页的数据，所以我们需要获取到第一屏的数据。这也就意味着双向绑定的数据操作在服务端是完全没有必要的

> 因为在服务端只会生成首评的内容，所以组件的生命周期函数中只有beforeCreate和created会在服务器端渲染(SSR)过程中被调用。

！！ 你应该避免在 beforeCreate 和 created 生命周期时产生全局副作用的代码，例如在其中使用 setInterval 设置 timer。

在客户端请求数据和在服务器端请求数据必须使用两个平台都可以使用的api。例如，axios 是一个 HTTP 客户端，可以向服务器和客户端都暴露相同的 API。

对于仅浏览器可用的 API，通常方式是，在「纯客户端(client-only)」的生命周期钩子函数中惰性访问(lazily access)它们。

 
## 源码结构

### 避免状态单例


> 当编写纯客户端(client-only)代码时，我们习惯于每次在新的上下文中对代码进行取值。但是，Node.js 服务器是一个长期运行的进程。当我们的代码进入该进程时，它将进行一次取值并留存在内存中。这意味着如果创建一个单例对象，它将在每个传入的请求之间共享。


因此，我们不应该直接创建一个应用程序实例，而是应该暴露一个可以重复执行的工厂函数，为每个请求创建新的应用程序实例

## 代码分割


应用程序的代码分割或惰性加载，有助于减少浏览器在初始渲染中下载的资源体积，可以极大地改善大体积 bundle 的可交互时间(TTI - time-to-interactive)。这里的关键在于，对初始首屏而言，"只加载所需"。



## 数据预取和状态

### 数据预取存储容器(Data Store)

在服务器端渲染(SSR)期间，我们本质上是在渲染我们应用程序的"快照"，所以如果应用程序依赖于一些异步数据，那么在开始渲染过程之前，需要先预取和解析好这些数据。

另一个需要关注的问题是在客户端，在挂载(mount)到客户端应用程序之前，需要获取到与服务器端应用程序完全相同的数据 - 否则，客户端应用程序会因为使用与服务器端应用程序不同的状态，然后导致混合失败。

为了解决这个问题，获取的数据需要位于视图组件之外，即放置在专门的数据预取存储容器(data store)或"状态容器(state container)）"中。首先，在服务器端，我们可以在渲染之前预取数据，并将数据填充到 store 中。此外，我们将在 HTML 中序列化(serialize)和内联预置(inline)状态。这样，在挂载(mount)到客户端应用程序之前，可以直接从 store 获取到内联预置(inline)状态。

### 带有逻辑配置的组件(Logic Collocation with Components)

在服务端预取数据的逻辑可以放在组件中，因为要加载的组件和要加载的数据是对应的。

在 entry-server.js 中，我们可以通过router.getMatchedComponents()获得与路由  相匹配的组件，如果组件暴露出 asyncData，我们就调用这个方法。然后我们需要将解析完成的状态，附加到渲染上下文(render context)中。

### 服务端数据预取

我们可以通过router.getMatchedComponents()这个方法来获取路由匹配的组件，如果组件暴露出asyncData，我们就调用这个方法。然后将解析完的状态，附加到渲染上下文(render context)中。我们在服务端预取数据之后需要同步到客户端，我们使用模版把服务端当前的状态保存在window对象中。这样客户端就可以同步服务端的状态了。

### 客户端数据预取

我猜测客户端数据预取指的是在服务端首屏渲染完毕之后，再处理访问其他的路由，我们不可能让服务端渲染每一个页面，一般来说只会渲染首屏页面，用于提高渲染速度和优化SEO。当用户访问其他的路由的时候，我们其实实在客户端独立渲染的，也就是说，我们选在已经把服务端的状态同步到客户端了，其他要做的就和我们平时写客户端程序是一样的。但是还有一个问题就是，我们再写ssr同构应用的时候我们肯定不想同一个请求写两遍，于是我们就需要在客户端的entry做手脚。

官方提供了两种解决方案，
1、把下一个路由页面的需要的数据全部获取到，然后在一起渲染。
2、在渲染到每一个组件之后，再去获取每一个组件独立的需要的数据。

两种方案各有可取之处。

### 客户端激活

指的是 Vue 在浏览器端接管由服务端发送的静态 HTML，使其变为由 Vue 管理的动态 DOM 的过程。

由于服务器已经渲染好了 HTML，我们显然无需将其丢弃再重新创建所有的 DOM 元素。相反，我们需要"激活"这些静态的 HTML，然后使他们成为动态的（能够响应后续的数据变化）。

如果你检查服务器渲染的输出结果，你会注意到应用程序的根元素上添加了一个特殊的属性：

```
<div id="app" data-server-rendered="true">
```

data-server-rendered 特殊属性，让客户端 Vue 知道这部分 HTML 是由 Vue 在服务端渲染的，并且应该以激活模式进行挂载。



在开发模式下，Vue 将推断客户端生成的虚拟 DOM 树(virtual DOM tree)，是否与从服务器渲染的 DOM 结构(DOM structure)匹配。如果无法匹配，它将退出混合模式，丢弃现有的 DOM 并从头开始渲染。在生产模式下，此检测会被跳过，以避免性能损耗。

一些需要注意的坑
使用「SSR + 客户端混合」时，需要了解的一件事是，浏览器可能会更改的一些特殊的 HTML 结构。例如，当你在 Vue 模板中写入：
```
<table>
  <tr><td>hi</td></tr>
</table>
```
浏览器会在 <table> 内部自动注入 <tbody>，然而，由于 Vue 生成的虚拟 DOM(virtual DOM) 不包含 <tbody>，所以会导致无法匹配。为能够正确匹配，请确保在模板中写入有效的 HTML。


## Bundle Renderer 指引

vue-server-renderer 提供一个名为 createBundleRenderer 的 API，用于处理此问题，通过使用 webpack 的自定义插件，server bundle 将生成为可传递到 bundle renderer 的特殊 JSON 文件。所创建的 bundle renderer，用法和普通 renderer 相同，但是 bundle renderer 提供以下优点：

> 内置的 source map 支持（在 webpack 配置中使用 devtool: 'source-map'）

> 在开发环境甚至部署过程中热重载（通过读取更新后的 bundle，然后重新创建 renderer 实例）

> 关键 CSS(critical CSS) 注入（在使用 *.vue 文件时）：自动内联在渲染过程中用到的组件所需的CSS。更多细节请查看 CSS 章节。

> 使用 clientManifest 进行资源注入：自动推断出最佳的预加载(preload)和预取(prefetch)指令，以及初始渲染所需的代码分割 chunk。

