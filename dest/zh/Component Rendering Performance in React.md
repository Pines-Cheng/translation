> 原文地址：[Component Rendering Performance in React](https://medium.com/modus-create-front-end-development/component-rendering-performance-in-react-df859b474adc)

React 因为性能好而被广为周知，但这并不意味着我们能够把这个当作是理所当然。让你的React应用更快的关键Tips之一就是优化你的 [render](https://facebook.github.io/react/docs/component-specs.html#render) 函数.

我曾经创建过一个简单的测试，来比较下面不同条件下的  `render()` 函数的速度：

- 无状态（函数）组件（stateless components） vs 有状态（基于class的）组件components
- 纯组件(Pure component)渲染 vs 无状态组件
- React 0.14 vs React 15 在development模式和production模式下的渲染性能

## 关键结论(TL;DR)

对于那些不耐烦，只是想阅读结果的人来说，这是以下测试中最重要的学习记录：

- 无状态组件Stateless (functional) components 并不比有状态组件 stateful (class)快
- React 15 的渲染性能大概比 0.14快25%
- 纯组件Pure components更快，因为使用shouldComponentUpdate
- development 模式下的渲染性能比production 模式下慢 2–8x倍
- React 15 development模式下的渲染性能比0.14慢 2x 倍 

很惊讶？

与每个基准一样，理解方法是弄懂结果的关键。让我们花一些时间来解释我在这里做了什么，为什么。

## 怎样进行性能测试

目标是创建一个非常简单的测试来迭代 render() 函数。我创建了包含三种组件之一的父组件：

- 一个无状态组件
- 一个有状态组件
- 一个纯粹的有状态组件，即shouldComponentUpdate 返回 false

```javascript
// Stateful component
class Stateful extends Component {
  render () {
    return <div>Hello Cmp1: stateful</div>;
  }
}
// Pure stateful with disabled updates
class Pure extends Component {
  shouldComponentUpdate() {
    return false;
  }
  render () {
    return <div>Hello Cmp2: stateful, pure, no updates</div>;
  }
}
// Stateless component
function Stateless() {
  return <div>Hello Cmp3: stateless</div>;
}
```

测试的组件很简单，并没有改变DOM。

顶层的组件将三种组件类型的每种循环渲染100,000次。同时使用浏览器自带的 [Performance.now](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now)功能，统计从初始渲染到最后渲染的时间来衡量渲染时间。遗憾的是，我们不能使用React官方提供的的Perf函数来统计，因为它们不能用在生产环境中。

虽然props始终被传递以确保更新，但目标组件保持渲染的数据相同。这样我们可以确保与纯组件的一致性。 DOM不会从这些组件中更新，从而保证其他API（例如布局呈现引擎）的干扰达到最小。

所有的JavaScript代码都运行在纯ES6中，没有转换步骤（没有转换为ES5

测试运行的浏览器为Chrome 51 (包含常用的插件，如一系列的 dev tools, 1Password, Pocket等), 一个纯粹的Chrome Canary 54 (没有插件，没有历史记录，完全全新的) 和 Firefox 46. OS X 10.11，主机环境为一台2.6 GHz Intel Core i7 处理器的MBP.所有呈现的数据都是取所有浏览器运行结果的平均值。

 [TJ Holowaychuk’s](https://medium.com/@tjholowaychuk) 的观点中，值得一提的是不安装任何插件的浏览器能够产生一个明显更好的结果。这就是我们使用一个纯粹的Chrome Canary配置的原因。无论怎样，我们普通的用户通常都会在浏览器安装几个插件，我们并不知道这个会导致多大的性能损失。

执行这些类型的基准测试时的精确度并不容易实现。有时测试会运行速度更慢或更快，导致结果产生偏差。这就是为什么我多次运行测试，并统计了所有的结果。您单次测试的结果可能会有所不同。

所有的 [源码](https://github.com/grgur/stateless-vs-pure) 都在Github上，欢迎check out. 每个框架对应一个文件夹，你能够在里面运行常见的 npm i && npm start命令。这些应用运行在不同的端口，因此他们能够被同时执行。 [readme 文件](https://github.com/grgur/stateless-vs-pure/blob/master/README.md) 提供了更详尽的说明。

我们已经有了这些前提，现在，让我们来讨论一下这些发现。

## 神话：无状态组件更快？

根据React 0.14和React 15，无状态（functional）组件与常规的基于类的状态组件一样快。

![image](https://cdn-images-1.medium.com/max/1600/1*QBk6Aa4zyaCtj-lcfrWl-w.png)

您会注意到，React 0.14中的测试显示无状态组件与状态组件性能有5％的差异，但我认为这是统计误差。

但是，当整个生命周期被剥离，且没有引用，没有状态管理，无状态组件为何不会更快？

根据 [Dan Abramov](https://medium.com/@dan_abramov)的说法，无状态组件内部封装在一个类中，当前并没有进行任何的优化，

> @ggrgur There is no “optimized” support for them yet because stateless component is wrapped in a class internally. It's same code path.

React团队已经[承诺](https://facebook.github.io/react/blog/2015/10/07/react-v0.14.html#stateless-functional-components)将对于无状态组件的优化，我确信在React的未来版本之一中将实现。

## React中最简单的性能技巧

抛开复杂构建优化问题，React应用程序性能调优的关键在于知道什么时候渲染，什么时候不渲染。

这里有一个例子：假设您正在创建一个下拉菜单，并希望通过设置状态来展开它expanded：true。这种情况下，你必须要渲染整个块，包括下拉列表中的所有列表项才能更改css值吗？千万不要这么干！

这种情况下，[shouldComponentUpdate](https://facebook.github.io/react/docs/component-specs.html#updating-shouldcomponentupdate) 是你最好的朋友， 赶紧用上它。如果你能够使用[shouldComponentUpdate](https://facebook.github.io/react/docs/component-specs.html#updating-shouldcomponentupdate)进行优化，就不要使用无状态组件。您可以使用Shallow Compare函数来简化流程，但如果这成为React组件核心功能的一部分，我也不会感到惊讶。

更新：从React 15.3开始，React.PureComponent是一个新的基类，可以替代使用Shallow Compare插件或Pure Render mixin的场景。

![image](https://cdn-images-1.medium.com/max/1600/1*gFmCRpaJ362W-zR5HP4shw.png)

最初的基准测试是在没有添加任何逻辑到render函数的情况下，进行了渲染性能的比较。而一旦添加了计算，Pure Components的优势就会更加明显。我们可以将其视为一种缓存的方法。
我这么直接原因之一是看到开发人员不关心改善渲染，因为虚拟DOM将在这方面做了一些处理。但是显然，在涉及到虚拟DOM的diff算法之前，应用的速度还有大量的提升空间。

我不想说你应该在所有的地方使用shouldComponentUpdate。添加大量的逻辑或在组件很少输出相同的HTML代码的地方使用它，就会增加不必要的复杂度。你应该知道这种生命周期方法的力量，并且合理的使用它。

阻止不必要的渲染只是其中一方面。上述结果告诉我们如何提高渲染性能？

## 优雅的Rendering

下图显示了render()函数对应用程序性能的影响。渲染是复杂操作的开始，最终导致优化的DOM更新。

![image](https://cdn-images-1.medium.com/max/1600/1*UGcD_NbwS4CxnRDc8XRNng.png)

render越简单，更新越快：

- 执行的JavaScript代码越少越好  — 特别是在移动网站/应用程序
- 返回更少的更改将有助于加快 virtual DOM 的计算
- 父组件 (container) 的render很可能会出发所有子孙组件的render()， 这意味着更多的计算。

![image](https://cdn-images-1.medium.com/max/1600/1*Y4zFnbujqSt-rKkFhACd7Q.gif)

以下是优化render阶段的几个提示：

- 如果可能，跳过render
- 在render函数外面，使用变量缓存开销大的计算
- … 或者将逻辑拆分进多个组件，然后有选择地管理render方法
- 尽可能的返回简单一点
- 保持render方法的纯粹 (使用函数式编程的思想)
- 保证state的扁平化，使用浅比较

正如您的应用程序可能在render阶段包含业务逻辑计算一样，React还添加了helpers来增强您的开发体验。让我们看看它们是如何影响性能的。

## 使用Production模式进行构建

React代码默认为development模式。令人惊讶的是，development环境下的渲染显着较慢。

![image](https://cdn-images-1.medium.com/max/1600/1*w5AgUaaW1e_w-s6-oDxapg.png)

在构建应用程序时将其指定为production模式的方法是定义环境变量NODE_ENV = production。当然，你只会在生产环境下用到这个，因为development 模式将提供更好的调试体验。

以下是您在Webpack配置中自动执行此变量的方法：

```javascript
module.exports = {
  plugins: [
    new webpack.DefinePlugin({
      ‘process.env’: {NODE_ENV: ‘”production”’}
    })
  ],
}
```

这样做不仅能够优化你的render性能，还能够提供更小的体积。

别忘了使用UglifyJS plugin剔除不必要的代码，下面是一个使用例子：

```javascript
plugins: [
  new webpack.optimize.UglifyJsPlugin({
    sourceMap: false,
    warnings: false,
  }),
],
```

我们看到React 15在development模式下与之前的版本相比要慢得多。他们在生产中的比较结果又如何呢？

## React 15 更快

React 15 [重要更新](https://facebook.github.io/react/blog/2016/04/07/react-v15.html) 是框架与DOM交互的核心变化。 innerHTML 被替换成 document.createElement，现代浏览器中更快捷的选择。
不再支持Internet Explorer 8可能意味着一些内部流程更加优化。
React 15在渲染过程中真的更快，但只有在构建模式为production时才能生效。development模式实际上相当慢，主要是因为包含了太多的帮助调试和优化代码的函数。
值得注意的是，这些发现是基于React 15与React-DOM 15的。React Native的development模式下结果可能会有显着差异。也许你可以运行类似的测试，并与社区分享结果。

## 写在最后

性能开始于在React应用程序中优化render block。该基准测试比较了创建和优化组件的三种方法。你得对他们的使用场景和优缺点了然于心。

可能还有许多其他的基准测试方法，因此绝对不要把你在这里学到的一切当作理所当然。如果您有其他想法欢迎贡献。

> Grgur是Modus Create的一名软件架构师，专门从事JavaScript性能，React JS和Sencha框架。如果这篇文章有帮助，也许他16年的从财富100强到政府和初创公司的经验可以帮助你的项目。
