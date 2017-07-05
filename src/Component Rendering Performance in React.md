> 原文地址：[Component Rendering Performance in React](https://medium.com/modus-create-front-end-development/component-rendering-performance-in-react-df859b474adc)

React is known for performance but it doesn’t mean we can take things for granted. One of the key performance tips for faster React applications is to optimize your [render](https://facebook.github.io/react/docs/component-specs.html#render) function.

I have created a simple test where I compared the speed of the `render()` function under several conditions:

- Stateless (functional) components vs stateful (class-based) components
- Pure component rendering vs stateless components
- React 0.14 vs React 15 rendering performance in development and production

## Key Takeaways (TL;DR)

For those who are impatient and just want to read the results, here’s the list of the most important learnings from this experiment:

- Stateless (functional) components are not any faster than the stateful (class)
- Rendering in React 15 is roughly 25% faster compared to 0.14
- Pure components are the fastest. Use shouldComponentUpdate
- Rendering in development mode is 2–8x slower than rendering in production
- Rendering in development with React 15 is about 2x slower than 0.14

Surprised?

As with every benchmark, understanding the methodology is the key to understanding the results. Let’s spend some time explaining what I did here and why.

## How Performance Was Tested

The goal was to create a very simple test that iterates over the render() function. I created the parent App component that contained one of the three types of components:

- A stateless component
- A stateful component
- A pure stateful component with shouldComponentUpdate returning false

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

Tested components are simple and do not change the DOM

Top level App component cycles through 100,000 renders for each of the three component types. Time to render was measured from the initial render to the last one using the browser’s native [Performance.now](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now) functionalities. I couldn’t use React’s wonderful Perf utilities because they don’t work in production.

While props were always passed to ensure update, target components kept the rendered data the same. That way we could ensure consistency with pure components. DOM is never updated from within these components to minimize interference with other APIs (e.g. layout rendering engine).

All of the JavaScript code ran in pure ES6 with no transpilation step(no transformation to ES5).

Tests were performed on Chrome 51 (with regular extensions such as various dev tools, 1Password, Pocket, etc), a clean Chrome Canary 54 (no plugins, no history, fresh) and Firefox 46. OS X 10.11 on a MBP sporting a 2.6 GHz Intel Core i7 processor made the host environment. All the numbers presented are average values from the runs in all browsers.

To [TJ Holowaychuk’s](https://medium.com/@tjholowaychuk) point, it’s worth mentioning that a browser with no plugins will offer significantly better results. This is why I used a clean Chrome Canary configuration. However, our common user will likely have a number of plugins installed and we won’t know how much of a performance drawback they will cause.

Precision when doing these types of benchmarks is never easy to achieve. Sometimes a test will run slower or faster, skewing results. That’s why I ran the tests several times and combined the results. Your results may vary.

The entire [source code](https://github.com/grgur/stateless-vs-pure) is in GitHub so please check it out. You’ll find a folder for each framework where you can run the usual npm i && npm start. The apps are on different ports too so they could be executed simultaneously. The [readme file](https://github.com/grgur/stateless-vs-pure/blob/master/README.md) will clearly point that out.

Now that we have this covered, let’s talk about the findings.

## Myth: Stateless Components are Faste

As of React 0.14 and React 15, stateless or functional components are just as fast as regular, class-based stateful components.

![image](https://cdn-images-1.medium.com/max/1600/1*QBk6Aa4zyaCtj-lcfrWl-w.png)

You’ll notice that the tests in React 0.14 show 5% difference in stateless vs stateful performance, but I attribute that to statistical error.

But how can stateless components not be faster when the entire lifecycle is stripped out, there are no refs, and there is no state to manage?

Stateless components are internally wrapped in a class without any optimizations currently applied, according to [Dan Abramov](https://medium.com/@dan_abramov).

> @ggrgur There is no “optimized” support for them yet because stateless component is wrapped in a class internally. It's same code path.

Optimizations to stateless components [have been promised](https://facebook.github.io/react/blog/2015/10/07/react-v0.14.html#stateless-functional-components) and I’m sure something will happen on that plan in one of the future versions of React.

## The Simplest Performance Trick in React

Complex build optimizations aside, the crucial step in performance tuning for React applications is knowing when to render. And when not to.

Here’s an example: let’s say you’re creating a drop-down menu and you want to expand it by setting state to expanded: true. Do you have to render the entire block including all the list items inside the drop-down just to change that css value? Absolutely not!

[shouldComponentUpdate](https://facebook.github.io/react/docs/component-specs.html#updating-shouldcomponentupdate) is your friend, so use it. Do not use stateless components if you can optimize with [shouldComponentUpdate](https://facebook.github.io/react/docs/component-specs.html#updating-shouldcomponentupdate). You can currently streamline the process with Shallow Compare function but I wouldn’t be surprised if this became part of the core functionality in React components. Does Smart Component or Pure Component ring a bell?

Update: as of React 15.3, React.PureComponent is a new base class that replaces the need to use Shallow Compare plugin or the Pure Render mixin.

![image](https://cdn-images-1.medium.com/max/1600/1*gFmCRpaJ362W-zR5HP4shw.png)

The original benchmarks compared rendering performance without any logic added to the render function. As soon as we add calculations the benefits of Pure components become even more apparent. We could think of it as a method of caching, too.
One of the reasons I made this so apparent is seeing developers not caring for improving rendering because virtual DOM is going to do the magic for me anyway. Obviously there’s plenty of room to make the app faster before even touching virtual DOM’s diff-ing capabilities.

I don’t want to say that you should use shouldComponentUpdate all over the place. Adding lots of logic or using it where components rarely output the same HTML code would just add the unnecessary burden. As with everything else, know the power of this lifecycle method and use it wisely.

Not rendering is just one side of the coin. What do the findings above tell us about improving rendering performance?

## Smarter Rendering
The charts above showed the impact of the render() function on your application’s performance. Render is the start of complex operations that eventually lead to optimized DOM updates.

![image](https://cdn-images-1.medium.com/max/1600/1*UGcD_NbwS4CxnRDc8XRNng.png)

The simpler the render, the faster the update:

- Less JavaScript code to process — particularly important for mobile sites/apps
- Returning fewer changes will help speed up virtual DOM calculation
- Render of a parent component (container) will likely trigger render() of all of it’s children. And grandchildren. That means more computations.

![image](https://cdn-images-1.medium.com/max/1600/1*Y4zFnbujqSt-rKkFhACd7Q.gif)

Here are a few tips for optimizing the render phase:

- Skip render if possible
- Cache expensive computations in variables outside render functions
- … or separate logic into multiple components and manage rendering selectively
- If possible, return early
- Keep render() slim (think functional programming)
- Keep comparisons shallow

Just as your app may contain business logic computations inside the render phase, React also adds helpers to enhance your development experience. Let’s see how they impact performance.


## Don’t Forget To Build for Production

By default, React sources are set to development mode. Not surprisingly, rendering in development environment is significantly slower.

![image](https://cdn-images-1.medium.com/max/1600/1*w5AgUaaW1e_w-s6-oDxapg.png)

The way to specify production mode when building your app is to define environment variable NODE_ENV=production. Of course, you want this only for production builds. Development counterparts will offer a much better debugging experience.

Here’s how you would go about automating this variable in your Webpack configuration:

```javascript
module.exports = {
  plugins: [
    new webpack.DefinePlugin({
      ‘process.env’: {NODE_ENV: ‘”production”’}
    })
  ],
}
```

This will not just optimize render performance, but will also result in a much smaller bundle size.

Don’t forget to use the unavoidable UglifyJS plugin that also eliminates dead code. Here’s an example of how you could use it:

```javascript
plugins: [
  new webpack.optimize.UglifyJsPlugin({
    sourceMap: false,
    warnings: false,
  }),
],
```

We saw how React 15 in development mode is much slower comparing to its predecessor. How do they compare in production?


## React 15 is Faster

One of the most [important updates](https://facebook.github.io/react/blog/2016/04/07/react-v15.html) in React 15 is the core change of how the framework interacts with DOM. innerHTML was replaced with document.createElement, a faster alternative for modern browsers.
Internet Explorer 8 no longer being supported likely means that some of the internal processes are more optimized.
React 15 truly is faster in the rendering process, but only when built for production. Development mode is actually quite a bit slower, mostly because of the plethora of functionalities that help debug and optimize code.
Note that these findings are based on React 15 with React-DOM 15. The values may be significantly different in React Native development. Maybe you could run a similar test and share results with the community.

## Conclusion

Performance starts with optimizing the render block in React applications. This benchmark compares the three approaches to creating and optimizing components. Know when to use each and where each excels.

There could be many other ways of benchmarking React performance so definitely don’t take everything you learned here for granted. If you have other ideas please contribute.

> Grgur is a software architect at Modus Create, specializing in JavaScript performance, React JS, and Sencha frameworks. If this post was helpful, maybe his 16 years of experience with clients ranging from Fortune 100 to governments and startups could help your project too.