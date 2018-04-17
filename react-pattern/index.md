#### 1.Redux导致的组件多余的渲染问题
请仔细阅读[React 组件间通讯](http://taobaofed.org/blog/2016/11/17/react-components-communication/)的文章的最后一个例子，最后的输出结果为:

![](https://img.alicdn.com/tfs/TB1tsBQNVXXXXaGaXXXXXXXXXXX-1190-216.png)

这是因为在最后一个定时器中是如下的代码:

```js
   setTimeout(() => {
      store.dispatch({
        type: 'child_2_1',
        data: 'bye'
      })
    }, 2000);
```
此时你必须了解redux的观察者模式，当你dispatch一个事件的时候，我们的Child_2与Child_2_1中subscribe的事件都会被执行，所以首先是Child_2_1直接进行了一次更新，然后接着由于Child_2的更新有发生了一次更新，所以Child_2_1总共在后面的dispatch中输出了两次，即渲染了两次。对于redux的观察者模式不了解的可以[查看redux原理分析](https://github.com/liangklfangl/redux-createstore)。同时，该文章给出了react兄弟组件通信的方法:即采用观察者模式或者redux(实际上也是观察者模式)来解决

#### 2.React开发中那些设计模式
(1)使用this.props.children来处理组件的低耦合。此时，父组件可以访问和读取子组件。而我们以前直接将Navigator组件写入到Header组件的方式一方面使得测试不容易(当然，在测试中可以使用[shallow-rendering](https://facebook.github.io/react/docs/test-utils.html#shallow-rendering)的方式来解决)，另一方面我们的Header和Navigator将会强耦合，对于一些不需要Navigator的Header组件来说就无法实现复用

(2)高阶组件可以很容易的实现逻辑的复用，对于一些重复的逻辑可以考虑使用[高阶组件](https://github.com/liangklfangl/high-order-reducer)来完成。下面的例子就为我们的初始组件添加了一个title属性：

```js
var config = require('path/to/configuration');
var enhanceComponent = (Component) =>
  class Enhance extends React.Component {
    render() {
      return (
        <Component
          {...this.state}
          {...this.props}
          title={ config.appTitle }
        \/>
      )
    }
  };
```
而这一切对于我们的Component来说是透明的，它只需要知道会接受到一个`title属性`即可，至于如何获取到，从哪里获取到这是他不需要关系的。而且高阶组件中也可以使用this.state或者this.props，前者在Enhance中直接定义，而后者是通过enhanceComponent方法对组件进行包装后传递的pros属性。高阶组件有一点需要注意，如下面的例子：

```js
var OriginalComponent = () => <p>Hello world.<\/p>;
class App extends React.Component {
  render() {
    return React.createElement(enhanceComponent(OriginalComponent));
  }
};
```
这段代码速度比较慢，每次都会清除OriginalComponent组件和其子组件的DOM和state状态，即完全重新渲染，因为enhanceComponent(OriginalComponent)每次都会给你返回一个不同的class，而根据React的component diff算法，组件本身的类型发生了改变，因此需要reRender。只要简单的修改如下:

```js
var OriginalComponent = () => <p>Hello world.<\/p>;
var EnhancedComponent = enhanceComponent(OriginalComponent);
class App extends React.Component {
  render() {
    return React.createElement(EnhancedComponent);
  }
};
```
此时render方法每次返回的都是同类型的EnhancedComponent。你可以查看[React的component diff](https://zhuanlan.zhihu.com/p/20346379?columnSlug=purerender)

(3)我们写的大部分的模块和组件都会存在依赖关系，特别是当树形的组件树存在的时候，父子关系的依赖就会出现了。因此，项目成功的一个关键就是如何去管理这些依赖关系。此时你应该要考虑到一个广为人知的设计模式，也就是
我们所说的:依赖注入。我们给出下面的例子：

```js
// Title.jsx
export default function Title(props) {
  return <h1>{ props.title }<\/h1>;
}
// Header.jsx
import Title from './Title.jsx';
export default function Header() {
  return (
    <header>
      <Title />
    <\/header>
  );
}
// App.jsx
import Header from './Header.jsx';
class App extends React.Component {
  constructor(props) {
    super(props);
    this.state = { title: 'React in patterns' };
  }
  render() {
    return <Header />;
  }
};
```
此时，假如你有一个需求，即你需要将“React in patterns”这个字符串传递给我们的Title组件。最直接的方法就是将属性从最顶层的App组件逐步传递到我们的Title组件。然而，当我们的属性很多或者组件嵌套的非常深的时候，那么很多中间的组件就需要处理那些他们根本不感兴趣的属性。此时，你可以考虑我们这里说的高阶组件的方法:

```js
// enhance.jsx
var title = 'React in patterns';
var enhanceComponent = (Component) =>
  class Enhance extends React.Component {
    render() {
      return (
        <Component
          {...this.state}
          {...this.props}
          title={ title }
        \/>
      )
    }
  };
// Header.jsx
import enhance from './enhance.jsx';
import Title from './Title.jsx';
var EnhancedTitle = enhance(Title);
//此时经过我们的enchance方法的处理，我们的Title组件会接受到一个title属性
export default function Header() {
  return (
    <header>
      <EnhancedTitle />
    <\/header>
  );
}
```

注意:此时我们的title属性并没有从Header组件传递过去，而是直接在我们的Title组件中通过高阶组件的形式进行了注入了。我们的title属性对于高阶组件本身来说是透明的，他根本不知道我们要给最终的Title组件传递title属性。这很好，但是这只是解决了一半的问题，我们虽然不需要将title属性通过组件树的形式传递下去了，但是我们的数据该如何到达enchance.jsx呢?
React引入了context的概念，每一个组件都可以访问我们的context属性。他就像是一个事件系统，但是他是为了数据而设计的。

```js
// a place where we'll define the context
var context = { title: 'React in patterns' };
class App extends React.Component {
  getChildContext() {
    return context;
  }
  ...
};
App.childContextTypes = {
  title: React.PropTypes.string
};

// a place where we need data
class Inject extends React.Component {
  render() {
    var title = this.context.title;
    ...
  }
}
Inject.contextTypes = {
  title: React.PropTypes.string
};
```

注意：我们需要完整的指定context对象的签名，即使用childContextType和getChildContext。如果任意一项没有指定，那么我们的context对象就是空对象。但是这似乎不太合理，我们可能需要在context中指定任意类型的数据。最佳实践告诉我们，context不应该仅仅是一个纯对象，而同时应该提供一个接口用于保存和获取数据，例如：

```js
// dependencies.js
export default {
  data: {},
  get(key) {
    return this.data[key];
  },
  register(key, value) {
    this.data[key] = value;
  }
}
```
因此，回到上面的例子，我们的App组件最终会是下面的形式:

```js
import dependencies from './dependencies';
dependencies.register('title', 'React in patterns');
class App extends React.Component {
  getChildContext() {
    return dependencies;
  }
  render() {
    return <Header />;
  }
};
App.childContextTypes = {
  data: React.PropTypes.object,
  get: React.PropTypes.func,
  register: React.PropTypes.func
};
```
而我们的Title组件通过context来获取到数据:

```js
// Title.jsx
export default class Title extends React.Component {
  render() {
    return <h1>{ this.context.get('title') }<\/h1>
  }
}
Title.contextTypes = {
  data: React.PropTypes.object,
  get: React.PropTypes.func,
  register: React.PropTypes.func
};
```
当然，我们并不想在任何需要使用context地方都声明一下contextTypes，而这个细节我们可以包裹到我们的高阶组件中。同样的，我们可以声明一个工具函数而来帮助我们操作context，而不是使用this.context.get('title')这种方式来获取我们需要的值，此时我们直接告诉高阶组件我们需要哪些数据，然后让他通过prop的形式传递给我们最终的组件，如下：

```js
// Title.jsx
import wire from './wire';
function Title(props) {
  return <h1>{ props.title }<\/h1>;
}
export default wire(Title, ['title'], function resolve(title) {
  return { title };
});
```

这个wire方法第一个参数是我们的React组件，第二个参数是所有已经注册的依赖的数组，第三个参数可以叫做mapper，是一个函数。这个函数会接受到我们保存在context中的值，然后返回给我们的是一个对象，这个对象中保存的是我们组件，如Title组件真实的prop属性。在这个例子中，我们只是传递了一个title的字符串，但是在一个大型的应用中很可能就是一个数据仓库，配置信息或者其他类型的信息。因此，我们要做到，传递给组件的prop属性是这个组件真实需要的，而不要使用无用的数据来污染组件。

下面是wire函数的签名：

```js
export default function wire(Component, dependencies, mapper) {
  class Inject extends React.Component {
    render() {
      var resolved = dependencies.map(this.context.get.bind(this.context));
      var props = mapper(...resolved);
      return React.createElement(Component, props);
    }
  }
  Inject.contextTypes = {
    data: React.PropTypes.object,
    get: React.PropTypes.func,
    register: React.PropTypes.func
  };
  return Inject;
};
```
Inject是一个高阶组件，他会访问context属性，然后获取哪些dependencies指定的属性。而我们的mapper方法会获取context数据，然后将它转化为组件的props.

很显然我们依赖注入的方法是使用了react的context特性，react官方并不建议使用react的context属性。但是对于这种依赖注入的方式原理是很好理解的，那些想要从context中访问数据的组件来说只要简单的使用wire方法进行包装即可。wire方法内部通过contextTypes来指定需要访问的数据。

当然，如果你想要使用context，那么你可以考虑使用模块系统。虽然，模块系统并不是为了React设计的，但是他也能实现同样的功能。我们知道，典型的模块系统有一个缓存机制，这在nodejs的文档中有明确的说明：

<pre>
  (1)当模块首次被加载的时候会被缓存，也就是说每次调用require('foo')都会得到完全相同的对象，只要这个模块对应于同一个文件。
  (2)多次调用require('foo');不会导致模块代码被多次执行,这是一个很重要的特性。利用这个特性，我们将会返回一个"部分完整"的对象(引用)，从而允许那些过渡的依赖被加载，即使这些过渡的依赖存在循环引用的关系。
</pre>

那么上面说的这些为什么能够处理我们的依赖注入呢？其实，每次我们导出一个对象的时候我们导出的其实是一个`单例对象`，其他的模块使用这个模块的时候，引入的对象也是同一个对象。这样的话，我们可以先注册我们的依赖，然后在其他文件中直接引入这个依赖。如下面的例子：

```js
var dependencies = {};
export function register(key, dependency) {
  dependencies[key] = dependency;
}
//fetch方法从全局变量dependencies获取数据
export function fetch(key) {
  if (dependencies[key]) return dependencies[key];
  throw new Error(`"${ key } is not registered as dependency.`);
}
export function wire(Component, deps, mapper) {
  return class Injector extends React.Component {
    constructor(props) {
      super(props);
      this._resolvedDependencies = mapper(...deps.map(fetch));
    }
    render() {
      return (
        <Component
          {...this.state}
          {...this.props}
          {...this._resolvedDependencies}
        />
      );
    }
  };
}
```
我们将依赖保存到`全局变量dependencies`(是我们模块的全局变量，而不是应用级别的全局变量)。同时，我们导出两个register和fetch方法来写和读取入口，就像实现对一个简单对象的getter/setter一样。然后，我们有一个wire函数，该函数接受到我们的React组件，然后返回一个高阶组件。在这个组件的构造函数中，我们会解析依赖，然后在后续对组件渲染的时候作为props传递给组件。我们看看如何使用上面这个方法：

```js
// app.jsx
import Header from './Header.jsx';
import { register } from './di.jsx';
//从模块加载register方法，其他方法都不加载，这种加载称之为“编译时加载”,也就是说ES6在编译时就能够完成模块编译，效率要比commonJS更高。此时，我们模块中的全局变量也会被引入
register('my-awesome-title', 'React in patterns');
//通过上面的方法在模块的全局变量dependencies中注册了一个属性
class App extends React.Component {
  render() {
    return <Header />;
  }
};
// -----------------------------------
// Header.jsx
import Title from './Title.jsx';
export default function Header() {
  return (
    <header>
      <Title />
    <\/header>
  );
}
// -----------------------------------
// Title.jsx
import { wire } from './di.jsx';
var Title = function(props) {
  return <h1>{ props.title }<\/h1>;
};
export default wire(Title, ['my-awesome-title'], title => ({ title }));
```
这个例子就是使用了模块系统来取代我们的React提供的context特性。其实际的做法就是引入了一个单独的文件可以对我们的应用的数据进行操作，并将数据经过处理后作为props传递给我们的组件进行渲染。当然，如果你需要让子组件可以通过register方法操作应用数据从而导致组件树进行更新，你依然需要对上面的register方法进行修改!

上面提供了context和事件系统的方式来实现依赖注入，你也可以考虑使用[inversifyJS](https://github.com/krasimir/react-in-patterns/tree/master/patterns/dependency-injection#injecting-with-the-help-of-a-build-process)，此处不再赘述。


(4)单向数据交流

单向数据流是React中一个很重要的概念。他的`主流思想是`：组件本身不修改他们接受到的props属性，他们仅仅是监听数据的变化，或者提供一个新的数据(用于更新prop数据,如通过父组件传递的callback)，但是他们不会直接更新props数据，这种组件就是我们常说的pure组件。而store中数据的更新是来自于另外一个机制，另外一个地方，而组件本身只会通过这个更新的数据来重新渲染而已。下面是React[官网](https://facebook.github.io/react/docs/components-and-props.html)的一句话：

*All React components must act like pure functions with respect to their props.*

我们日常开发中常常使用的方式为：父组件将回调传递给子组件，而子组件通过调用该callback导致父组件(或者最顶级的组件)的state发生了变化，从而导致整个组件树重新渲染。其实我们常说的redux数据流管理也是同样的道理，即通过修改了最顶层的组件的state导致了组件树的重新渲染。

下面给出Switcher的例子：

```js
class Switcher extends React.Component {
  constructor(props) {
    super(props);
    this.state = { flag: false };
    this._onButtonClick = e => this.setState({ flag: !this.state.flag });
  }
  render() {
    return (
      <button onClick={ this._onButtonClick }>
        { this.state.flag ? 'lights on' : 'lights off' }
      <\/button>
    );
  }
};
// ... and we render it
class App extends React.Component {
  render() {
    return <Switcher />;
  }
};
```
在这个例子中，我们的组件本身是有数据的。Switcher是唯一一个地方知道我们flag属性的值的。下面我们展示一个store的例子，此时不仅Switcher我们的store本身也是知道flag的：

```js
var Store = {
  _flag: false,
  set: function(value) {
    this._flag = value;
  },
  get: function() {
    return this._flag;
  }
};

class Switcher extends React.Component {
  constructor(props) {
    super(props);
    this.state = { flag: false };
    this._onButtonClick = e => {
      this.setState({ flag: !this.state.flag }, () => {
        this.props.onChange(this.state.flag);
      });
    }
  }
  render() {
    return (
      <button onClick={ this._onButtonClick }>
        { this.state.flag ? 'lights on' : 'lights off' }
      <\/button>
    );
  }
};

class App extends React.Component {
  render() {
    return <Switcher onChange={ Store.set.bind(Store) } \/>;
  }
};
```
这个例子的的数据交流就是双向的，我们点击按钮的时候，React组件的状态发生变化导致重新渲染，这是可以理解的。但是点击按钮的时候，我们也会更新store的内容，因此我们就需要一种机制保证store和组件本身维护状态的一致(组件本身的状态通过state来维护，比如其他地方ajax请求导致store也变化了，那么我们就需要同步这种变化，如广播)。此时我们的数据流状态如下:

<pre>
    User's input
     |
  Switcher <-------> Store
                      ^ |
                      | |
                      | |
                      | v
    Service communicating
    with our backend
</pre>

而我们的单向数据流就解决了这个问题。他去除了多状态，而只是维护store这一种状态。为了实现这种效果，我们需要修改我们的store对象，使得我们可以订阅store的改变：

```js
var Store = {
  _handlers: [],
  _flag: '',
  onChange: function(handler) {
    this._handlers.push(handler);
  },
  set: function(value) {
    this._flag = value;
    this._handlers.forEach(handler => handler())
  },
  get: function() {
    return this._flag;
  }
};
```
此时，我们需要对我们的App组件进行处理，每次store发生变化的时候都重新渲染我们的组件：

```js
class App extends React.Component {
  constructor(props) {
    super(props);
    Store.onChange(this.forceUpdate.bind(this));
  }
  render() {
    return (
      <div>
        <Switcher
          value={ Store.get() }
          onChange={ Store.set.bind(Store) } />
      </div>
    );
  }
};
```
注意：此处我们使用了[forceUpdate](https://facebook.github.io/react/docs/react-component.html)，但是我们不推荐这样做，使用forceUpdate会使得当前组件直接跳过shouldComponentUpdate。但是所有的子组件的生命周期函数调用是正常的，包括shouldComponentUpdate，React只有在DOM更新的时候才会进行重新渲染。一种推荐的方式就是[通过state来维持一个key](https://stackoverflow.com/questions/30626030/can-you-force-a-react-component-to-rerender-without-calling-setstate)，每次key发生了变化，那么组件肯定会更新。

此时我们的Switcher组件变得非常简单，我们不需要维持内部的状态了:

```js
class Switcher extends React.Component {
  constructor(props) {
    super(props);
    this._onButtonClick = e => {
      this.props.onChange(!this.props.value);
    }
  }
  render() {
    return (
      <button onClick={ this._onButtonClick }>
        { this.props.value ? 'lights on' : 'lights off' }
      <\/button>
    );
  }
}
```
我们这种方式的好处在于：我们的组件本身只是store状态的一种反映。我们只是将React组件作为views层，而且我们只需要在一个地方来管理我们的状态。此时我们的数据流是如下的形式：

<pre>
Service communicating
with our backend
    ^
    |
    v
  Store <-----
    |        |
    v        |
Switcher ---->
    ^
    |
    |
User input
</pre>
正如你能看到的，我们的数据流是单向的，我们不需要同步系统的两个部分，即UI和store。单向数据流不仅仅用于React,他是一种有用的模式，该模式会使得你的应用本身变得非常易于维护。

(5)在React组件的构造函数中调用bind方法

以前我们定义组件的onClick时候通常采用下面的方法：

```js
class Switcher extends React.Component {
  constructor(props) {
    super(props);
    this.state = { name: 'React in patterns' };
  }
  render() {
    return (
     <button onClick={ this._handleButtonClick.bind(this) }>
     //这里不能是onClick={ this._handleButtonClick }，因为此时无法在_handleButtonClick中访问this，所以要么在这里调用bind方法要么在constructor中调用bind
      click me
    <\/button>
    );
  }
  _handleButtonClick() {
    console.log(`Button is clicked inside ${ this.state.name }`);
    // leads to
    // Uncaught TypeError: Cannot read property 'state' of null
  }
};
```
这种情况下bind方法会不断调用，因为我们会多次渲染我们的button,即触发它的render方法(如组件更新)。所以，好的方法应该是将bind放到constructor方法中：

```js
class Switcher extends React.Component {
  constructor(props) {
    super(props);
    this.state = { name: 'React in patterns' };
    this._buttonClick = this._handleButtonClick.bind(this);
  }
  render() {
    return (
      <button onClick={ this._buttonClick }>
        click me
      <\/button>
    );
  }
  _handleButtonClick() {
    console.log(`Button is clicked inside ${ this.state.name }`);
  }
};
```

#### 3.React性能优化
##### 3.1 Stateless Component
 (1)对于Stateless组件来说，顾名思义，其不会维持自己的State状态，组件的展示只是与props相关，只有props发生了改变，不管其值有没有发生变化，该组件`都会重新渲染`。比如下面的例子:

```js
class ChangeName extends React.Component{
    state ={
     name:'liangklfangl'
    }
    changeName = ()=>{
       this.setState({name:'liangklfangl'});
    }
    render(){
      return (
       <div>
        <HelloComponent name={this.state.name}\/>
        <button onClick={this.changeName}\/>
      <\/div>
    )
    }
  }
  //组件的第二个参数是context
  function HelloComponent(props, /* context */) {
   console.log('HelloComponent');
     return <div>Hello {props.name}<\/div>
  }
      ReactDOM.render(
        <ChangeName/>,
        document.getElementById('example')
      );
```
此时当你点击按钮的时候，虽然我们的HelloComponent的name的值依然是"liangklfangl"，但是我们的HelloComponent也会被重新渲染一次。

(2)无状态组件中没有this，所以其函数不需要bind绑定。在无状态组件中不要使用其他函数，只要props发生了变化，其没有SCU(shouldComponentUpdate)，所以都会更新，因此会在函数内部不断创建新的函数，从而浪费内存空间。比如下面的例子:

```js
import React from "react";
const HelloWorld = ({name}) =>{
  const sayHi = (event)=>{
    alert(`Hi ${name}`);
  }
  return (
    <div>
     <a href="#" onClick={sayHi}>Say Hi<\/a>
    <\/div>)
}
```
上面的例子在无状态组件HelloWorld中定义了一个函数，那么当我们的HelloWorld组件每次被重新渲染的时候都会重新实例化sayHi函数，从而导致内存浪费。一个好的解决方案是:`将这些函数放到无状态组件之外，通过传递属性值的方式来实现，这将给性能带来很大提升`。

(3)React组件仅仅是一个无状态函数，React在渲染的时候也省掉了将“组件类”实例化的过程，这也是为什么组件没有this的原因。无状态组件没有实现组件的生命周期方法，例如 componentDidMount, componentWillUnmount等。他只是单纯的根据props来展示UI，如果要它根据窗口大小来做出响应，可以引入`容器组件`来处理；同时，我们的React无状态组件是不包含ref的，因为在 React 调用到无状态组件的方法之前，是没有一个实例化的过程的，因此也就没有所谓的 "ref"

##### 3.2 React.PureComponent
React.PureComponent的出现并不是说我们的组件不会维持它自己的state，而是由于React官方为了简化开发者对于`state和props是否发生变化的判断`，而为我们自动提供了SCU方法，从而当state和props变化以后才会更新，否则不会更新组件。这样可以有效的减少组件重新渲染的次数，提升性能。你可以[查看这个例子](../react-context/Context-Problemmatic/readme.md)，该例子展示了由于React.PureComponent的state和props没有发生变化，从而导致其自身以及所有的子组件都不会被重新渲染。这是React.PureComponent和context同时存在导致的潜在问题。

但是，React.PureComponent有一个`潜在的问题`，也就是他对于props和state是否发生变化的比较是通过浅复制来完成的，这对于组件是否需要更新的判断可能会存在问题。因为引用数据，比如数组或者对象，如果引用没有变化，那么组件是不会重新渲染的。比如下面的例子:

```js
handleClick() {
  let {items} = this.state
  items.push('new-item')
  this.setState({ items })
}

render() {
  return (
    <div>
      <button onClick={this.handleClick} \/>
      //如果ItemList是React.PureComponent组件，那么点击按钮组件不会更新
      <ItemList items={this.state.items} \/>
    <\/div>
  )
}
```

此时我们应该使用数组的concat而不是push，从而创建一个新的对象，这样组件才会正常更新。比如下面的例子:

```js
handleClick() {
  this.setState(prevState => ({
    words: prevState.items.concat(['new-item'])
  }));
}
```

我们再给出一个经常遇到的代码示例:

```js
class App extends React.Component{
  constructor(props) {
      super(props)
      this.update = this.update.bind(this)
  }
  update(e) {
      this.props.update(e.target.value)
  }
  render() {
      return <MyInput onChange={this.update} \/>
   }
}
```
如果我们的MyInput是纯组件，那么我们必须采用在constructor中bind的方式，而不是采用如下的形式，因为后者每次都会产生一个新的函数,比如我们的App组件如要重新渲染，此时我们的MyInput组件的onChange函数将会是一个新的函数实例。那么如果MyInput是一个React.PureComponent，那么他的效果就无法正常发挥出来，`因为每次props都会发生变化，从而重新渲染`。

```js
class App extends React.Component{
  constructor(props) {
      super(props)
  }
  update(e) {
      this.props.update(e.target.value)
  }
  render() {
      return <MyInput onChange={()=>this.update()} \/>
  }
}
```

同时，请谨记：纯组件忽略重新渲染时(SCU返回false)，[不仅会影响它本身，而且会影响它的所有子元素](../react-context/Context-Problematic/readme.md)。所以，使用React.PureComponent的最佳情况就是展示组件，它既没有子组件，也没有依赖应用的全局状态。而这一切都基于下面的事实:

<pre>
  在JSX中，任何包含子元素(child elements)的组件， shallowEqual检查总会返回false。
</pre>

这也就是说：如果我们的React.PureComponent本身包含子组件(注意：我们的button,div等不是组件，组件必须是继承于React.Component或者React.PureComponent等自己声明的类或者函数)，那么其shallowEqual总是返回false，那么组件总是更新。这样并不能通过使用React.PureComponent来减少组件重新渲染的次数。因此，纯组件仅仅适用于展示组件而不是容器组件。


##### 3.3 Stateless Component vs React.PureComponent
通过上面的分析，你应该知道Stateless Component是没有state只有props的，没有ref，没有组件实例也就是没有this，没有组件的其他声明周期方法，他是一个pure函数。而React.PureComponent只是React官方给我们自动添加了SCU方法而已，他有state，有生命周期，有ref等其他无状态组件没有的行为。但是必须注意，React.PureComponent对于SCU的判断是基于浅复制的，浅复制的，浅复制的!!!!!





参考资料:

[React 组件间通讯](http://taobaofed.org/blog/2016/11/17/react-components-communication/)

[React.js in patterns](http://krasimirtsonev.com/blog/article/react-js-in-design-patterns)

[react in patterns](https://github.com/krasimir/react-in-patterns/tree/master/patterns/higher-order-components)

[Higher order components should be wrapped before render()](https://github.com/krasimir/react-in-patterns/issues/12)

[无状态组件(Stateless Component) 与高阶组件](http://www.jianshu.com/p/63569386befc)

[九个你忽略的React 无状态组件的优势](http://www.zcfy.cc/article/1980)

[无状态组件(Stateless Component) 与高阶组件](http://www.jianshu.com/p/63569386befc)

[在React.js中使用PureComponent的重要性和使用方式](http://www.open-open.com/lib/view/1484466792204)
