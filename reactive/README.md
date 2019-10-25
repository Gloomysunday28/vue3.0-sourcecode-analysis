# Vue3.0数据响应系统分析(主要针对于reactive)

## Vue3.0与Vue2.x的数据响应机制
Vue3.0采用了ES6的Proxy来进行数据监听

  优点:

    1. 对对象进行直接监听, 可以弥补Object.defineProperty无法监听新增删除属性的短板

    2. 无需在遍历对象进行设置监听函数

    3. 可以适用于Array, 不需要再分成两种写法

  缺点:

    1. 兼容性不足, 导致目前IE11无法使用
  
  ## 源码导读

  在分析源码之前，我们需要得知几个变量先:
  ```javascript
    rawToReactive:
      类型: <WeakMap>
      值: {
        原始对象: Proxy对象
      }

    reactiveToRaw:
      类型: <WeakMap>
      值: {
        Proxy对象: 原始对象
      }

    targetMap: 
      类型: <WeakMap>
      值: {
         原始对象: new Map([key, new Set([effect])]) // key是原始对象里的属性， 值为该key改变后会触发的一系列的函数， 比如渲染、computed
      }
  ```

  首先我们来看一下reactive函数

  ```javascript
    export function reactive(target: object) {
    // if trying to observe a readonly proxy, return the readonly version.
    if (readonlyToRaw.has(target)) {
      return target
    }
    // target is explicitly marked as readonly by user
    if (readonlyValues.has(target)) {
      return readonly(target)
    }
    return createReactiveObject(
      target,
      rawToReactive,
      reactiveToRaw,
      mutableHandlers,
      mutableCollectionHandlers
    )
  }

  ```
首先我们检测了原始对象是否是只读的代理对象， 紧接着又检测了是否是只读对象， 原因是readonly对象是不允许进行修改编辑， 所以是不需要进行响应处理, 接下来， 主要的响应系统都在createReactiveObject里， 下面对该函数进行分析

```javascript
  function createReactiveObject(
    target: any,
    toProxy: WeakMap<any, any>, // { originObj: proxyObj }
    toRaw: WeakMap<any, any>, // { proxyObj: originObj }
    baseHandlers: ProxyHandler<any>,
    collectionHandlers: ProxyHandler<any>
  ) {
    if (!isObject(target)) {
      if (__DEV__) {
        console.warn(`value cannot be made reactive: ${String(target)}`)
      }
      return target
    }
    // target already has corresponding Proxy
    let observed = toProxy.get(target) 
    if (observed !== void 0) { 
      return observed
    }
    
    // target is already a Proxy
    if (toRaw.has(target)) {
      return target
    }

    // only a whitelist of value types can be observed.
    if (!canObserve(target)) {
      return target
    }

    const handlers = collectionTypes.has(target.constructor)
      ? collectionHandlers
      : baseHandlers
    observed = new Proxy(target, handlers)
    toProxy.set(target, observed)
    toRaw.set(observed, target)
    
    if (!targetMap.has(target)) {
      targetMap.set(target, new Map())
    }
    return observed
  }
```
首先对于类型进行了判断，若不是对象形式则会报错，并且不进行响应式处理
其次对是否已经处理过该对象进行了判断，由于Proxy对象与原对象已经不是同一个指针，所以Vue对两个对象进行了分别的判断

canObserve判断是否符合以下四个条件
```
  对象符合以下配置
   * 1. 不是Vue实例
   * 2. 不是虚拟DOM
   * 3. 是属于Object|Array|Map|Set|WeakMap|WeakSet其中一种
   * 4. 不存在于nonReactiveValues
```
handlers这里判断了两种情况:
  
  1. 若是Map|Set|WeakMap|WeakSet的一种则采用collectionHandlers
  2. 否则采用baseHandlers

最后进行原对象的代理处理, 并且绑定了两者的关系, 在这里我们看见了targtMap的绑定， 这个WeakMap对于**数据响应**起到了很关键的作用，我们下面会讲到，先看下面, 紧接着就返回了代理对象

接下来我们来看下代理对象handler的处理

由于百分之99的处理都是由baseHandlers来处理，那么我们接下来就针对这个handlers进行分析

```javascript
  export const mutableHandlers: ProxyHandler<any> = {
    get: createGetter(false),
    set,
    deleteProperty,
    has,
    ownKeys
  }
```
很简单的一个赋值， 我们从get函数开始分析
```javascript
  function createGetter(isReadonly: boolean) {
    return function get(target: any, key: string | symbol, receiver: any) {
      const res = Reflect.get(target, key, receiver)
      if (typeof key === 'symbol' && builtInSymbols.has(key)) {
        return res
      }
      if (isRef(res)) {
        return res.value
      }
      
      track(target, OperationTypes.GET, key)
      return isObject(res)
        ? isReadonly
          ? // need to lazy access readonly and reactive here to avoid
            // circular dependency
            readonly(res)
          : reactive(res)
        : res
    }
  }
```
首先获取了该属性的值，然后我们看见Vue对Symbol的一些类型进行分辨， 若是符合条件则直接返回, 接下来这里划重点，要考, track函数对于数据响应起到了至关重要的作用, 我们来看下**track**函数源码是怎么写的

```javascript
  export function track(
    target: any,
    type: OperationTypes,
    key?: string | symbol
  ) {
    if (!shouldTrack) {
      return
    }
    const effect = activeReactiveEffectStack[activeReactiveEffectStack.length - 1]

    if (effect) {
      if (type === OperationTypes.ITERATE) {
        key = ITERATE_KEY
      }
      let depsMap = targetMap.get(target) 
      if (depsMap === void 0) {
        targetMap.set(target, (depsMap = new Map()))
      }
      
      let dep = depsMap.get(key!)
      if (dep === void 0) {
        depsMap.set(key!, (dep = new Set()))
      }
      if (!dep.has(effect)) {
        dep.add(effect)
        effect.deps.push(dep)
        if (__DEV__ && effect.onTrack) {
          effect.onTrack({
            effect,
            target,
            type,
            key
          })
        }
      }
    }
  }
```
首先这里定义了一个shouldTrack, 这个变量是用来控制调用生命周期的时候的开关，防止触发多次

获取targetMap里该对象各个属性的值, 若没有，则进行数据初始化new Set, 并且将effect添加到了该集合里, 这里我们看见了 不止dep添加了, effect也添加了, 这里是有原因的，我们等下进行分析

看到这里, 相信大家都明白了track函数使用进行数据依赖采集的, 以便于后面数据更改能够触发对应的函数

**接下来我们分析下set函数**
```javascript
  function set(
    target: any,
    key: string | symbol,
    value: any,
    receiver: any // proxy对象
  ): boolean {
    value = toRaw(value)
    const hadKey = hasOwn(target, key)
    const oldValue = target[key]
    if (isRef(oldValue) && !isRef(value)) {
      oldValue.value = value
      return true
    }
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    if (target === toRaw(receiver)) { // 判断更改对象是否是
      /* istanbul ignore else */
      if (__DEV__) {
        const extraInfo = { oldValue, newValue: value }
        if (!hadKey) {
          trigger(target, OperationTypes.ADD, key, extraInfo)
        } else if (value !== oldValue) {
          trigger(target, OperationTypes.SET, key, extraInfo)
        }
      } else {
        if (!hadKey) {
          trigger(target, OperationTypes.ADD, key)
        } else if (value !== oldValue) {
          trigger(target, OperationTypes.SET, key)
        }
      }
    }
    return result
  }
```
首先一开始也是对于各个类型进行分析并且处理对应的情况, 我们主要看一下trigger函数
```javascript
  export function trigger(
    target: any, // 原始对象
    type: OperationTypes, // 判断是替换还是增加等等操作
    key?: string | symbol, // 对象属性
    extraInfo?: any
  ) {
    const depsMap = targetMap.get(target) // new Map()
    // console.log('depsMap1', depsMap);
    if (depsMap === void 0) {
      // never been tracked
      return
    }
    const effects = new Set<ReactiveEffect>()
    const computedRunners = new Set<ReactiveEffect>()
    if (type === OperationTypes.CLEAR) {
      // collection being cleared, trigger all effects for target
      depsMap.forEach(dep => {
        addRunners(effects, computedRunners, dep)
      })
    } else {
      // console.log(key);
      // schedule runs for SET | ADD | DELETE
      if (key !== void 0) {
        addRunners(effects, computedRunners, depsMap.get(key))
      }
      // also run for iteration key on ADD | DELETE
      if (type === OperationTypes.ADD || type === OperationTypes.DELETE) {
        const iterationKey = Array.isArray(target) ? 'length' : ITERATE_KEY
        addRunners(effects, computedRunners, depsMap.get(iterationKey))
      }
    }
    const run = (effect: ReactiveEffect) => {
      scheduleRun(effect, target, type, key, extraInfo)
    }
    // Important: computed effects must be run first so that computed getters
    // can be invalidated before any normal effects that depend on them are run.
    computedRunners.forEach(run)
    effects.forEach(run)
  }
```
在trigger函数里我们看到了两个集合变量, effects与computedRunners, 两个集合针对两种类型进行数据采集我么往下看

引入眼帘的估计就是**addRunners**, 我们猜测下 这个addRunners顾名思义应该就是添加执行任务, 下面看下源码
```javascript
  function addRunners(
    effects: Set<ReactiveEffect>,
    computedRunners: Set<ReactiveEffect>,
    effectsToAdd: Set<ReactiveEffect> | undefined // Effects集合
  ) {
    // console.log('effectsToAdd', effectsToAdd);
    if (effectsToAdd !== void 0) {
      effectsToAdd.forEach(effect => {
        if (effect.computed) {
          computedRunners.add(effect)
        } else {
          effects.add(effect)
        }
      })
    }
  }
```
effectsToAdd就是track函数里添加的对象属性的值 new Set, 用于收集依赖的,
根据effect是否是计算属性来分别添加到不同的集合下, 回到trigger函数里, 我们看见了effects与computedRunners进行遍历执行, 那么我们在分析下具体的**scheduleRun**函数
```javascript
  function scheduleRun(
    effect: ReactiveEffect,
    target: any,
    type: OperationTypes,
    key: string | symbol | undefined,
    extraInfo: any
  ) {
    if (__DEV__ && effect.onTrigger) {
      effect.onTrigger(
        extend(
          {
            effect,
            target,
            key,
            type
          },
          extraInfo
        )
      )
    }
    
    if (effect.scheduler !== void 0) {
      effect.scheduler(effect)
    } else {
      effect()
    }
  }
```
这里我们看见了Vue对effect进行了两种情况的判断, 首先判断了effect.scheduler是否存在, 若存在则使用scheduler来调用effect, 不存在则进行直接调用, 那么scheduler到底是什么呢？ 这里的scheduler就是Vue的性能优化点，放入队里里, 等到miscroTask里进行调用, 熟悉Vue2.x的同学都知道nextTick函数, 这个scheduler可以看做就是调用了nextTick函数

我们来看下effect具体是什么
```javascript
  const effect: ReactiveEffect = function effect(...args: any[]): any {
    return run(effect as ReactiveEffect, fn, args)
  }

  function run(effect: ReactiveEffect, fn: Function, args: any[]): any {
    if (!effect.active) {
      return fn(...args)
    }

    if (activeReactiveEffectStack.indexOf(effect) === -1) {
      cleanup(effect)
      // 初始化mount的时候会执行effect函数， 当前effect是componentEffect, 也就是渲染函数, 此时由于去获取了变量数据，也就是触发了get函数，get函数会触发track函数, track函数就是用来收集effect, 
      try {
        activeReactiveEffectStack.push(effect)
        return fn(...args)
      } finally {
        activeReactiveEffectStack.pop()
      }
    }
  }
```
effect实际上就是运行了run函数, 我们看下run函数的运行, 在运行之前会先cleanup, 这里我们就要返回之前所说的track函数, 大家还记得track函数里, 不只dep添加了effect, effect也同时添加了dep吗? 原因就在这里, cleanup需要用到
```javascript
  function cleanup(effect: ReactiveEffect) {
    const { deps } = effect
    // console.log('deps', deps);
    if (deps.length) {
      for (let i = 0; i < deps.length; i++) {
        deps[i].delete(effect)
      }
      deps.length = 0
    }
  }
```
该函数清空了dep里所有的依赖, 那么胆大心细的同学会发现一个问题:

  在track函数里已经添加了effect, 那么为什么在这里要重新清除掉所有的依赖呢?

  理论上看起来是个很鸡肋的操作, 但是实际上Vue已经考虑了全方面, 试想一个场景:
  A组件与B组件是通过v-if来控制展示, 当A组件首先渲染之后, 所对应的的数据就会采集对应的依赖, 此时更改v-if条件, 渲染了B组件, 若是B组件此时更改了A组件里的变量, 若是A组件的依赖没有被清除掉, 那么会产生不必要的依赖调用, 所以Vue要事先清除掉所有的依赖, 确保依赖始终是最新的


分析到这我们已经清楚了Vue3.0的数据响应究竟是如何了！

# 总结
Vue3.0从根本上解决了Vue2.x数据响应系统留下的短板, 但是兼容性上还存在问题,采用尤大一句话, IE11百足之虫,死而不僵, 暂时还不能完全抛弃IE11, 希望后期能有新的突破！