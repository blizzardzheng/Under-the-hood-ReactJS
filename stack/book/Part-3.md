## Part 3

[![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3.svg)](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3.svg)

<em>3.0 第 3 部分 (点击查看大图)</em>

### 挂载

 `componentMount` 方法是我们整个系列中极其重要的一个板块。如图，我们关注`ReactCompositeComponent.mountComponent`(1) 方法

如果你还记得，前文提到过 **组件树的入口组件** 是 `TopLevelWrapper` 组件 ( React 底层内部类 )。 我们准备挂载它。由于它实际上是一个空的包装器，调试起来非常枯燥并且对实际的流程而言并没有任何影响，所以我们跳过这个组件从他的孩子组件开始分析。

把组件挂载到组件树上的过程就是先挂载父亲组件，然后他的孩子组件，然后他的孩子的孩子组件，依次类推。可以肯定，当 `TopLevelWrapper` 挂载后，他的孩子组件 (用来管理 `ExampleApplication` 的组件`ReactCompositeComponent` ) 也会在同一阶段注入.

现在我们回到步骤 (1) 观察这个方法的内部实现，有一些重要行为会发生，接下来让我们深入研究这些重要行为。

### 构造 instance 和 updater

从 `transaction.getUpdateQueue()` 结果返回的步骤 (2) 方法  `updater` 实际上就是  `ReactUpdateQueue`  模块。 那么为什么需要在这里构造它呢？因为我们正在研究的类 `ReactCompositeComponent` 是一个全平台的共用的类，但是 `updater` 却依赖于平台环境而不尽相同，所以我们在这里根据不同的平台动态的构造它。

然而，我们现在并不马上需要这个 `updater` ，但是你要记住它是非常重要的，因为它很快就会应用于非常知名的组件内更新方法**`setState`** 。

事实上在这个过程中，不仅仅`updater` 被构造，组件实例（你的自定义组件）也获得了继承的`props`, `context`, and `refs`.

我们来看下面的代码:

```javascript
// \src\renderers\shared\stack\reconciler\ReactCompositeComponent.js#255
// These should be set up in the constructor, but as a convenience for
// simpler class abstractions, we set them up after the fact.
inst.props = publicProps;
inst.context = publicContext;
inst.refs = emptyObject;
inst.updater = updateQueue;
```

因此，你才可以通过一个实例从你的代码中获得 `props` ，比如 `this.props`.

### 创建 ExampleApplication 实例

通过调用步骤 (3) 的方法  `_constructComponent` 以及几个构造方法，最终创建了  `new ExampleApplication()` 。这就是我们代码中构造方法第一次被执行的时机，当然也是我们的代码第一次实际接触到 React 的生态系统，很棒。

### Perform initial mount

接着我们研究步骤 (4) ，第一个即将发生的行为是  `componentWillMount`  (当然仅当它被定义) 的调用。这是我们遇到的第一个生命周期钩子函数。当然，在下面一点你会看到  `componentDidMount` 函数, 只不过这时由于它不能马上执行，而是被注入了一个事务队列中。他会在挂载系列操作执行完毕后执行。当然你也可能在 `componentWillMount ` 内部添加` setState` ，在这种情况下 `state` 会被重新计算但`render`并不会被执行。(这是合理的，因为这时候组件还没有被挂载)

官方文档的解释也证明这一点:

> `componentWillMount()` is invoked immediately before mounting occurs. It is called before `render()`, therefore setting state in this method will not trigger a re-rendering.

以下的代码使我们更自信

```javascript
// \src\renderers\shared\stack\reconciler\ReactCompositeComponent.js#476
if (inst.componentWillMount) {
    //..
    inst.componentWillMount();

    // When mounting, calls to `setState` by `componentWillMount` will set
    // `this._pendingStateQueue` 不会触发重渲染
    if (this._pendingStateQueue) {
        inst.state = this._processPendingState(inst.props, inst.context);
    }
}
```

True. Well, but when `state` is recalculated, we call the `render` method. Yes, exactly the one which we specify in our components! So, one more touch of ‘our’ code.

Alright, the next thing is to create a React component instance. Erm... what, again? Seems we’ve already seen this `this._instantiateReactComponent`(5) call, right? That’s true, but that time we instantiated a `ReactCompositeComponent` for our `ExampleApplication` component. Now, we are going to create VDOM instances for its child based on the element we got from the `render` method. For our exact case, the render method returns `div`, so the VDOM representation for it is `ReactDOMComponent`. When the instance is created, we call `ReactReconciler.mountComponent` again, but this time as `internalInstance`, we pass a newly created instance of `ReactDOMComponent`.

And, call `mountComponent` on it…

### 很好，我们已经完成了第三部分

Let’s recap how we got here. Let's look at the scheme one more time, then remove redundant less important pieces, and it becomes this:

[![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3-A.svg)](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3-A.svg)

<em>3.1 Part 3 simplified (clickable)</em>

And we should probably fix spaces and alignment as well:

[![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3-B.svg)](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3-B.svg)

<em>3.2 Part 3 simplified & refactored (clickable)</em>

Nice. In fact, that’s all that happens here. So, we can take the essential value from *Part 3* and use it for the final `mounting` scheme:

[![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3-C.svg)](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3-C.svg)

<em>3.3 Part 3 essential value (clickable)</em>

And then we're done!


[To the next page: Part 4 >>](./Part-4.md)

[<< To the previous page: Part 2](./Part-2.md)


[Home](../../README.md)
