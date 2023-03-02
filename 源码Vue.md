## new Vue()

```js
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'
import type { GlobalAPI } from 'types/global-api'

function Vue(options) {
  if (__DEV__ && !(this instanceof Vue)) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
	// 调用 this._init()方法  ==> 找它
  this._init(options)
}

// mixin相关的东西
//@ts-expect-error Vue has function type
initMixin(Vue)
//@ts-expect-error Vue has function type
stateMixin(Vue)
//@ts-expect-error Vue has function type
eventsMixin(Vue)
//@ts-expect-error Vue has function type
lifecycleMixin(Vue)
//@ts-expect-error Vue has function type
renderMixin(Vue)

export default Vue as unknown as GlobalAPI
```

## beforeCreated 和 created

```js
export function initMixin (Vue: Class<Component>) {
  // 在原型上添加 _init 方法
  Vue.prototype._init = function (options?: Object) {
    // 保存当前实例
    const vm: Component = this
    // 合并配置
    if (options && options._isComponent) {
      // 把子组件依赖父组件的 props、listeners 挂载到 options 上，并指定组件的$options
      initInternalComponent(vm, options)
    } else {
      // 把我们传进来的 options 和当前构造函数和父级的 options 进行合并，并挂载到原型上
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    vm._self = vm
		// 初始化生命周期
		// 初始化实例的属性、数据：$parent, $children, $refs, $root, _watcher...等
    initLifecycle(vm) 
		/*初始化事件*/
    initEvents(vm) // 初始化事件：$on, $off, $emit, $once
		/*初始化render*/
    initRender(vm) // 初始化渲染： render, mixin
    callHook(vm, 'beforeCreate') // 调用生命周期钩子函数
		// resolve injections before data/props 
    initInjections(vm) // 初始化 inject 
    initState(vm) // 初始化组件数据：props, data, methods, watch, computed
		// resolve provide after data/props
    initProvide(vm) // 初始化 provide 
    callHook(vm, 'created') // 调用生命周期钩子函数

    if (vm.$options.el) {
      /*挂载组件*/
      // 如果传了 el 就会自动调用 $mount 进入模板编译和挂载阶段
      // 如果没有传就需要手动执行 $mount 才会进入下一阶段
      vm.$mount(vm.$options.el)

	 // =>  执行完所有初始化的操作后，开始挂载Vue实例到dom上
    }
  }
}
```

##  $mount()

```js
Vue.prototype.$mount = function (
	// el 是字符串或者dom元素
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
	// 提示不能把body/html作为挂载点, 开发环境下给出错误提示
  if (el === document.body || el === document.documentElement) {
    __DEV__ &&
      warn(
        `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
      )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
	// 解析 template/el 并且转为render函数，
/*处理模板templete，编译成render函数，render不存在的时候才会编译template，否则优先使用render*/
  if (!options.render) {
    let template = options.template
		// 02  去看配置项中有没有配置 template 属性
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (__DEV__ && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (__DEV__) {
          warn('invalid template option:' + template, this)
        }
        return this
      }
		// 03 看配置项中有没有el 
    } else if (el) {
      // @ts-expect-error
      template = getOuterHTML(el)
    }
    if (template) {
      /* istanbul ignore if */
      if (__DEV__ && config.performance && mark) {
        mark('compile')
      }
/* 将template编译成render函数，这里会有render以及staticRenderFns两个返回，
这是vue的编译时优化，static静态不需要在VNode更新时进行patch，优化性能*/
      const { render, staticRenderFns } = compileToFunctions(
        template,
        {
          outputSourceRange: __DEV__,
          shouldDecodeNewlines,
          shouldDecodeNewlinesForHref,
          delimiters: options.delimiters,
          comments: options.comments
        },
        this
      )
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
      if (__DEV__ && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  return mount.call(this, el, hydrating)
}
```

## mountComponent

```js
export function mountComponent(
  vm: Component,
  el: Element | null | undefined,
  hydrating?: boolean
): Component {
  vm.$el = el
	// 判断有没有render函数，如果没有某些情况下会提示
  if (!vm.$options.render) {
    // @ts-expect-error invalid type
		// 省略~~~~~
  }
	// 调用生命周期钩子函数
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (__DEV__ && config.performance && mark) {
    updateComponent = () => {
			// 省略 --- 性能标记相关代码
    }
  } else {
    updateComponent = () => {
// 调用 _update 对 render 返回的虚拟 DOM 进行 patch（也就是 Diff )到真实DOM，这里是首次渲染
      vm._update(vm._render(), hydrating)
    }
  }

	// 监听组件的变化，如果更新了
  const watcherOptions: WatcherOptions = {
    before() {
			// 如果组件处于挂载状态，并且没有被销毁		
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }

  if (__DEV__) {
    watcherOptions.onTrack = e => callHook(vm, 'renderTracked', [e])
    watcherOptions.onTrigger = e => callHook(vm, 'renderTriggered', [e])
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
	// 为当前组件设置Watcher观察监听，看是否发生变化
  new Watcher(
    vm,
    updateComponent,
    noop,
    watcherOptions,
    true /* isRenderWatcher */
  )
  hydrating = false

  // flush buffer for flush: "pre" watchers queued in setup()
  const preWatchers = vm._preWatchers
  if (preWatchers) {
    for (let i = 0; i < preWatchers.length; i++) {
      preWatchers[i].run()
    }
  }

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true
	// 调用生命周期钩子函数
    callHook(vm, 'mounted')
  }
  return vm
}
```

## beforeDestroy 和 destroyed

```js
Vue.prototype.$destroy = function () {
    const vm: Component = this
    if (vm._isBeingDestroyed) {
      return
    }
    callHook(vm, 'beforeDestroy')
    vm._isBeingDestroyed = true
    // remove self from parent
    const parent = vm.$parent
		// 如果父级存在，并且父级没有在被销毁，且不是抽象组件
    if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
			// 从父级中删除当前组件
      remove(parent.$children, vm)
    }
    // teardown scope. this includes both the render watcher and other
    // watchers created
    vm._scope.stop()
    // remove reference from data ob
    // frozen object may not have observer.
		// 删除数据对象的引用
    if (vm._data.__ob__) {
      vm._data.__ob__.vmCount--
    }
    // call the last hook...
		// 更新组件销毁状态
    vm._isDestroyed = true
    // invoke destroy hooks on current rendered tree
		// 删除实例的虚拟 DOM
    vm.__patch__(vm._vnode, null)
    // fire destroyed hook
    callHook(vm, 'destroyed')
    // turn off all instance listeners.
    vm.$off()
    // remove __vue__ reference
    if (vm.$el) {
      vm.$el.__vue__ = null
    }
    // release circular reference (#6759)
    if (vm.$vnode) {
      vm.$vnode.parent = null
    }
  }
```

