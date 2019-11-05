<font color=red>3213</font>


# Vue初始化渲染

## 源码导读
设计到的目录:

  1. src/packages/vue src/packages/runtime-dom  入口文件
  2. src/packages/runtime-core 渲染核心代码
  3. src/packages/compiler-core 编译模板

## 走进代码， 体验前所未有的舒适！
### 入口
```javascript
  // src/packages/vue/src/index.ts
  import { compile, CompilerOptions } from '@vue/compiler-dom'
  import * as runtimeDom from '@vue/runtime-dom'
  import { registerRuntimeCompiler, RenderFunction } from '@vue/runtime-dom'

  function compileToFunction(
    template: string,
    options?: CompilerOptions
  ): RenderFunction {
    const { code } = compile(template, {
      hoistStatic: true,
      ...options
    })
    
    return new Function('Vue', code)(runtimeDom) as RenderFunction
  }

  registerRuntimeCompiler(compileToFunction)

  export { compileToFunction as compile }
  export * from '@vue/runtime-dom'

  if (__BROWSER__ && __DEV__) {
    console[console.info ? 'info' : 'log'](
      `You are running a development build of Vue.\n` +
        `Make sure to use the production build (*.prod.js) when deploying for production.`
    )
  }
```
从上面我们得知@vue/runtime-dom已经被引入，并且执行了全局活动对象, 这个runtime-dom很关键，让我们看下这里究竟做了什么

### runtime-dom
```javascript
  import { createRenderer } from '@vue/runtime-core'
  ...
  const { render, createApp } = createRenderer<Node, Element>({
    patchProp, // DOM属性设置， 分为事件、样式、属性、props
    ...nodeOps // DOM操作的原生方法
  })
  ...
```
createRenderer函数定义了多个对于渲染有作用的函数，我们来剖析一下主要的几个函数
### runtime-core
我们先看返回的两个函数
```javascript
  function createRenderer(...) {
    ...
    return {
      render,
      createApp: createAppAPI(render)
    }
  }

  function createAppAPI<HostNode, HostElement>(
    render: RootRenderFunction<HostNode, HostElement>
  ): () => App<HostElement> {
    return function createApp(): App {
      const app = {
        ...,
        mount(
          rootComponent: Component,
          rootContainer: string | HostElement,
          rootProps?: Data
        ): any {
          if (!isMounted) {
            ...
            render(vnode, rootContainer)
            isMounted = true
            return vnode.component!.renderProxy
          } else if (__DEV__) {
            warn(
              `App has already been mounted. Create a new app instance instead.`
            )
          }
        },
      }
    }
  }
```
在调用createApp会获取到一系列的方法, 例如mixin, use, mount等等,由于是渲染, 我们就单独讲讲<code>mount</code>函数, 我们看到会先调用render函数, render函数是外部传递进来的, 它被定义在createRenderer函数里
```javascript
  function render(vnode: HostVNode | null, rawContainer: HostElement | string) {
    ...
    if (vnode == null) {
      if (container._vnode) {
        unmount(container._vnode, null, null, true)
      }
    } else {
      patch(container._vnode || null, vnode, container)
    }
    flushPostFlushCbs()
    container._vnode = vnode
  }
```
我们先将patch函数， flushPostFlushCbs()在patch之后再讲
```javascript
  function patch(
    n1: HostVNode | null, // null means this is a mount
    n2: HostVNode, // 虚拟DOM
    container: HostElement,
    anchor: HostNode | null = null,
    parentComponent: ComponentInternalInstance | null = null,
    parentSuspense: HostSuspenseBoundary | null = null,
    isSVG: boolean = false,
    optimized: boolean = false
  ) {
    ...
    processComponent(
      n1,
      n2,
      container,
      anchor,
      parentComponent,
      parentSuspense,
      isSVG,
      optimized
    )
  }

  function processComponent(
    n1: HostVNode | null,
    n2: HostVNode,
    container: HostElement,
    anchor: HostNode | null,
    parentComponent: ComponentInternalInstance | null,
    parentSuspense: HostSuspenseBoundary | null,
    isSVG: boolean,
    optimized: boolean
  ) {
      ...
      mountComponent(
        n2,
        container,
        anchor,
        parentComponent,
        parentSuspense,
        isSVG
      )
  }

  function mountComponent(
    initialVNode: HostVNode,
    container: HostElement,
    anchor: HostNode | null,
    parentComponent: ComponentInternalInstance | null,
    parentSuspense: HostSuspenseBoundary | null,
    isSVG: boolean
  ) {
    ...
    // resolve props and slots for setup context
    const propsOptions = (initialVNode.type as Component).props
    resolveProps(instance, initialVNode.props, propsOptions)
    resolveSlots(instance, initialVNode.children)

    // setup stateful logic
    if (initialVNode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
      setupStatefulComponent(instance, parentSuspense)
    }
    ...
  }
```
我把每个函数重要的部分给列举出来，其余的省略掉了，我们可以看到，patch函数执行一路到mountComponent才是真正关键的地方, <code>mountComponent</code>里我们看到首先对props进行了处理， 再进行了slots处理， 之后进行了setupStatefulComponent调用
```javascript
  function setupStatefulComponent(
    instance: ComponentInternalInstance,
    parentSuspense: SuspenseBoundary | null
  ) {
    const Component = instance.type as ComponentOptions
    // 1. create render proxy
    instance.renderProxy = new Proxy(instance, PublicInstanceProxyHandlers)
    // 2. create props proxy
    // the propsProxy is a reactive AND readonly proxy to the actual props.
    // it will be updated in resolveProps() on updates before render
    const propsProxy = (instance.propsProxy = readonly(instance.props))
    // 3. call setup()
    const { setup } = Component
    if (setup) {
      const setupContext = (instance.setupContext =
        setup.length > 1 ? createSetupContext(instance) : null)

      currentInstance = instance
      currentSuspense = parentSuspense
    // console.log(11123123);

      const setupResult = callWithErrorHandling(
        setup,
        instance,
        ErrorCodes.SETUP_FUNCTION,
        [propsProxy, setupContext]
      )
      // console.log('setup', setup);
      currentInstance = null
      currentSuspense = null
        // console.log('setupResult', setupResult);
      if (
        setupResult &&
        isFunction(setupResult.then) &&
        isFunction(setupResult.catch)
      ) {
        if (__FEATURE_SUSPENSE__) {
          // async setup returned Promise.
          // bail here and wait for re-entry.
          instance.asyncDep = setupResult
        } else if (__DEV__) {
          warn(
            `setup() returned a Promise, but the version of Vue you are using ` +
              `does not support it yet.`
          )
        }
        return
      } else {
        handleSetupResult(instance, setupResult, parentSuspense)
      }
    } else {
      finishComponentSetup(instance, parentSuspense)
    }
  }
```
首先将组件实例进行代理对象处理,赋值给实例下的renderProxy, 接下来对setup进行判断用户是否定义了setup, **注意这里, 我们可以看见源码并没有对setup类型进行判断**, 但是vue会把它默认当做是函数来使用, 所以这里我们必须设定为函数, **不然代码会报错, 但是不影响渲染**,  我们根据大部分场景setup返回的是一个函数或者是响应数据来走, 此时我们会执行<font color=#00ffff>handleSetupResult</font>
```javascript
  function handleSetupResult(
    instance: ComponentInternalInstance,
    setupResult: unknown,
    parentSuspense: SuspenseBoundary | null
  ) {
    if (isFunction(setupResult)) {

      // setup returned an inline render function
      instance.render = setupResult as RenderFunction
    } else if (isObject(setupResult)) {
      if (__DEV__ && isVNode(setupResult)) {
        warn(
          `setup() should not return VNodes directly - ` +
            `return a render function instead.`
        )
      }
      // setup returned bindings.
      // assuming a render function compiled from template is present.
      instance.renderContext = reactive(setupResult)
    } else if (__DEV__ && setupResult !== undefined) {
      warn(
        `setup() should return an object. Received: ${
          setupResult === null ? 'null' : typeof setupResult
        }`
      )
    }

    finishComponentSetup(instance, parentSuspense)
  }
```
Vue首先判断setup返回的是否是一个函数，如果是的话那么将组件实例下的render函数赋值成该函数， 若不是，Vue会再判断是否是一个对象，该对象不能是虚拟DOM, 但是不会阻止渲染，最后将该对象进行响应处理并且赋值给组件实例下的<font color=#00ffff>renderContext</font>, 这个renderContext是一个挺重要的变量，后面会讲到， 最后再执行
<font color=red>finishComponentSetup</font>

```javascript
  finishComponentSetup(
    instance: ComponentInternalInstance,
    parentSuspense: SuspenseBoundary | null
  ) {
    const Component = instance.type as ComponentOptions
    if (!instance.render) {
      if (Component.template && !Component.render) {
        if (compile) {
          Component.render = compile(Component.template, {
            onError(err) {}
          })
          console.log('Component', Component.render);
        } else if (__DEV__) {
          warn(
            `Component provides template but the build of Vue you are running ` +
              `does not support on-the-fly template compilation. Either use the ` +
              `full build or pre-compile the template using Vue CLI.`
          )
        }
      }
      if (__DEV__ && !Component.render) {
        warn(
          `Component is missing render function. Either provide a template or ` +
            `return a render function from setup().`
        )
      }
      instance.render = (Component.render || NOOP) as RenderFunction
    }

    // support for 2.x options
    if (__FEATURE_OPTIONS__) {
      currentInstance = instance
      currentSuspense = parentSuspense
      applyOptions(instance, Component)
      currentInstance = null
      currentSuspense = null
    }

    if (instance.renderContext === EMPTY_OBJ) {
      instance.renderContext = reactive({})
    }
}

```
我们可以看到初始化渲染时, 组件实例是不具备render函数， 这里会判断用户定义的组件配置是否具有template同时不具备render函数, 满足条件后Vue会对**template进行解析成AST树, 转换代码并且生成代码, 最后会形成**
```javascript
  function render() {
    with(this) {
      return 'template所对应的模板'
    }
  }
```
最后组件实例下的render函数会被赋值成解析出来的render函数或者是空函数, 接着Vue3.0会去兼容2.x的写法, 调用了<font color=red>applyOptions</font>方法

```javascript
  function applyOptions(
    instance: ComponentInternalInstance,
    options: ComponentOptions,
    asMixin: boolean = false
  ) {
    ...
    if (!asMixin) {
      callSyncHook('beforeCreate', options, ctx, globalMixins)
      // global mixins are applied first
      applyMixins(instance, globalMixins)
    }

    // extending a base component...
    if (extendsOptions) {
      applyOptions(instance, extendsOptions, true)
    }
    // local mixins
    if (mixins) {
      applyMixins(instance, mixins)
    }
    ...

    if (!asMixin) {
      callSyncHook('created', options, ctx, globalMixins)
    } 

    if (beforeMount) {
      onBeforeMount(beforeMount.bind(ctx))
    }
    if (mounted) {
      onMounted(mounted.bind(ctx))
    }
    ...
  }

  function callSyncHook(
    name: 'beforeCreate' | 'created',
    options: ComponentOptions,
    ctx: any,
    globalMixins: ComponentOptions[]
  ) {
    callHookFromMixins(name, globalMixins, ctx)
    const baseHook = options.extends && options.extends[name]
    if (baseHook) {
      baseHook.call(ctx)
    }
    const mixins = options.mixins
    if (mixins) {
      callHookFromMixins(name, mixins, ctx)
    }
    const selfHook = options[name]
    if (selfHook) {
      selfHook.call(ctx)
    }
  }
```
在这里我们可以看见setup是在beforeCreate之前执行的, Vue会对该组件所有相关的组件进行调用生命周期, 顺序是这样的:
1. 全局mixin
2. extends
3. 组件mixin
4. 自身

接下来进行资源合并处理（data, computed, methods, inject）, 这里我们就说到刚才说到的重要变量renderContext, **它不只存放了setup返回的, 并且存储了data, methods, inject里的变量**, 

setupStatefulComponent至此讲完了

### 返回mountComponent函数
```javascript
  function mountComponent() {
    ...
    setupRenderEffect(
      instance,
      parentSuspense,
      initialVNode,
      container,
      anchor,
      isSVG
    )
  }

  function setupRenderEffect(
    instance: ComponentInternalInstance,
    parentSuspense: HostSuspenseBoundary | null,
    initialVNode: HostVNode,
    container: HostElement,
    anchor: HostNode | null,
    isSVG: boolean
  ) {
    // create reactive effect for rendering
    let mounted = false
    instance.update = effect(function componentEffect() {
      if (!mounted) {
        const subTree = (instance.subTree = renderComponentRoot(instance))
        // beforeMount hook
        if (instance.bm !== null) {
          invokeHooks(instance.bm)
        }
        patch(null, subTree, container, anchor, instance, parentSuspense, isSVG)
        initialVNode.el = subTree.el
        // mounted hook
        if (instance.m !== null) {
          queuePostRenderEffect(instance.m, parentSuspense)
        }
        mounted = true
      } else {
        // updateComponent
        // This is triggered by mutation of component's own state (next: null)
        ...
      }
    }, __DEV__ ? createDevEffectOptions(instance) : prodEffectOptions)
  }
```
mountComponent执行完setupStatefulComponent后还会执行setupRenderEffect函数， 我们来看下该函数的作用, 首先定义了一个变量mounted并且初始化为false, 其次定义了组件实例下的update方法，这里使用到了effect，也就是数据响应里讲到的依赖, 如果不是Computed，effect的第一个参数函数会被执行一次，此时会判断是否已经挂载了， 若没有， 则调用beforeMount与mounted声明周期，**由于生命周期可以是数组形式，所以Vue内部会以数组形式来进行调用**, <code>不知道大家有没有使用过onMounted与onBeforeMount方法，不管两个谁定义在前，都是按照bm -> m</code>, 这里其实也解释了为什么是这么调用的
