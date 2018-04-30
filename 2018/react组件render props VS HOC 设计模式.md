---
title: react组件render props VS HOC 设计模式
category: 前端
cdn: 'header-on'
tags: [react,HOC,Render Props]
---

React的[设计模式](https://levelup.gitconnected.com/react-component-patterns-ab1f09be2c82)有很多种，比如无状态组件/表现型组件，有状态组件/容器型组件，render模式组件，高阶组件等等。本文主要介绍react的render模式与HOC设计模式，并通过实际案例进行比较。

### render props模式
> The Render Props是一种在不重复代码的情况下共享组件间功能的方法。如下所示：

```js
<DataProvider render={data => (
  <h1>Hello {data.target}</h1>
)}/>
```
通过使用prop来定义呈现的内容，组件只是注入功能，而不需要知道它如何应用于UI。render prop 模式意味着用户通过定义单独组件来传递prop方法，来指示共享组件应该返回的内容。

Render Props 的核心思想是，通过一个函数将class组件的state作为props传递给纯函数组件
```js
import React from 'react';

const SharedComponent extends React.Component {
  state = {...}
  render() {
    return (
      <div>
        {this.props.render(this.state)}
      </div>
    );
  }
}

export default SharedComponent;
```

```this.props.render()```是由另外一个组件传递过来的。为了使用以上组件，我们可以进行下面的操作：
```js
import React from 'react';
import SharedComponent from 'components/SharedComponent';

const SayHello = () => (
  <SharedComponent render={(state) => (
    <span>hello!,{...state}</span>
  )} />
);
```
{this.props.render(this.state)}这个函数，将其state作为参数传入其的props.render方法中，调用时直接取组件所需要的state即可。
render props模式最重要的是它返回一个react元素，比如我将上面的render属性改名，依然有效。

```js
import React from 'react';

const SharedComponentWithGoofyName extends React.Component {
  render() {
    return (
      <div>
        {this.props.wrapThisThingInADiv()}
      </div>
    );
  }
}

const SayHelloWithGoofyName = () => (
  <SharedComponentWithGoofyName wrapThisThingInADiv={() => (
    <span>hello!</span>
  )} />
);
```

### HOC设计模式
React的高阶组件主要用于组件之间共享通用功能而不重复代码的模式（也就是达到DRY模式）。

高阶组件实际是一个函数。 HOC函数将组件作为参数并返回一个新的组件。它将组件转换为另一个组件并添加额外的数据或功能。

> 高阶组件在React生态链技术中经常用到，对读者较为熟悉的,比如Redux中的connect，React Router中的withRouter等。

常见的高阶组件如下所示：
```js
import React from 'react';

const withSecretToLife = (WrappedComponent) => {
  class HOC extends React.Component {
    render() {
      return (
        <WrappedComponent
          secretToLife={42}
          {...this.props}
        />
      );
    }
  }
    
  return HOC;
};

export default withSecretToLife;
```
已知secretToLife为42，有一些组件需要共享这个信息，此时创建了SecretToLife的HOC，将它作为prop传递给我们的组件。
```js
import React from 'react';
import withSecretToLife from 'components/withSecretToLife';

const DisplayTheSecret = props => (
  <div>
    The secret to life is {props.secretToLife}.
  </div>
);

const WrappedComponent = withSecretToLife(DisplayTheSecret);

export default WrappedComponent;
```
此时，WrappedComponent只是DisplayTheSecret的增强版本，允许我们访问secretToLife属性。

### Render Props与HOC模式实例对比
#### HOC Example
```js
import React from 'react';

const withStorage = (WrappedComponent) => {
  class HOC extends React.Component {
    state = {
      localStorageAvailable: false, 
    };
  
    componentDidMount() {
       this.checkLocalStorageExists();
    }
  
    checkLocalStorageExists() {
      const testKey = 'test';

      try {
          localStorage.setItem(testKey, testKey);
          localStorage.removeItem(testKey);
          this.setState({ localStorageAvailable: true });
      } catch(e) {
          this.setState({ localStorageAvailable: false });
      } 
    }
  
    load = (key) => {
      if (this.state.localStorageAvailable) {
        return localStorage.getItem(key); 
      }
      
      return null;
    }
    
    save = (key, data) => {
      if (this.state.localStorageAvailable) {
        localStorage.setItem(key, data);
      }
    }
    
    remove = (key) => {
      if (this.state.localStorageAvailable) {
        localStorage.removeItem(key);
      }
    }
    
    render() {
      return (
        <WrappedComponent
          load={this.load}
          save={this.save}
          remove={this.remove}
          {...this.props}
        />
      );
    }
  }
    
  return HOC; 
}

export default withStorage;
```
在withStorage中，使用componentDidMount生命周期函数来检查checkLocalStorageExists函数中是否存在localStorage。

local，save，remove则是来操作localStorage的。现在我们创建一个新的组件，将其包裹在HOC组件中，用于显示相关的信息。由于获取信息的API调用需要很长时间，我们可以假设这些值一旦设定就不会改变。我们只会在未保存值的情况下进行此API调用。 然后，每当用户返回页面时，他们都可以立即访问数据，而不是等待我们的API返回。
```js
import React from 'react';
import withStorage from 'components/withStorage';

class ComponentNeedingStorage extends React.Component {
  state = {
    username: '',
    favoriteMovie: '',
  }

  componentDidMount() {
    const username = this.props.load('username');
    const favoriteMovie = this.props.load('favoriteMovie');
    
    if (!username || !favoriteMovie) {
      // This will come from the parent component
      // and would be passed when we spread props {...this.props}
      this.props.reallyLongApiCall()
        .then((user) => {
          this.props.save('username', user.username) || '';
          this.props.save('favoriteMovie', user.favoriteMovie) || '';
          this.setState({
            username: user.username,
            favoriteMovie: user.favoriteMovie,
          });
        }); 
    } else {
      this.setState({ username, favoriteMovie })
    }
  }

  render() {
    const { username, favoriteMovie } = this.state;
    
    if (!username || !favoriteMovie) {
      return <div>Loading...</div>; 
    }
    
    return (
      <div>
        My username is {username}, and I love to watch {favoriteMovie}.
      </div>
    )
  }
}

const WrappedComponent = withStorage(ComponentNeedingStorage);

export default WrappedComponent;
```

在封装组件的componentDidMount内部，首先尝试从localStorage中获取，如果不存在，则异步调用，将获得的信息存储到localStorage并显示出来。

#### The Render Props Exapmle
```js
import React from 'react';

class Storage extends React.Component {
    state = {
      localStorageAvailable: false, 
    };
  
    componentDidMount() {
       this.checkLocalStorageExists();
    }
  
    checkLocalStorageExists() {
      const testKey = 'test';

      try {
          localStorage.setItem(testKey, testKey);
          localStorage.removeItem(testKey);
          this.setState({ localStorageAvailable: true });
      } catch(e) {
          this.setState({ localStorageAvailable: false });
      } 
    }
  
    load = (key) => {
      if (this.state.localStorageAvailable) {
        return localStorage.getItem(key); 
      }
      
      return null;
    }
    
    save = (key, data) => {
      if (this.state.localStorageAvailable) {
        localStorage.setItem(key, data);
      }
    }
    
    remove = (key) => {
      if (this.state.localStorageAvailable) {
        localStorage.removeItem(key);
      }
    }
    
    render() {
      return (
        <span>
          this.props.render({
            load: this.load,
            save: this.save,
            remove: this.remove,
          })
        </span>
      );
    } 
}
```
Storage组件内部与HOC的withStorage较为类似，不同的是Storage不接受组件为参数，并且返回this.props.render。

```js
import React from 'react';
import Storage from 'components/Storage';

class ComponentNeedingStorage extends React.Component {
  state = {
    username: '',
    favoriteMovie: '',
    isFetching: false,
  }

  fetchData = (save) => {
    this.setState({ isFetching: true });
    
    this.props.reallyLongApiCall()
      .then((user) => {
        save('username', user.username);
        save('favoriteMovie', user.favoriteMovie);

        this.setState({
          username: user.username,
          favoriteMovie: user.favoriteMovie,
          isFetching: false,
        });
      }); 
  }

  render() {
    return (
      <Storage
        render={({ load, save, remove }) => {
          const username = load('username') || this.state.username;
          const favoriteMovie = load('favoriteMovie') || this.state.username;
      
          if (!username || !favoriteMovie) {
            if (!this.state.isFetching) {
              this.fetchData(save);               
            }

            return <div>Loading...</div>; 
          }
      
          return (
            <div>
              My username is {username}, and I love to watch {favoriteMovie}.
            </div>
          );
        }}
      />
    )
  }
}
```
对于ComponentNeedingStorage组件来说，利用了Storage组件的render属性传递的三个方法，进行一系列的数据操作，从而展示相关的信息。

### render props VS HOC模式
总的来说，render props其实和高阶组件类似，就是在puru component上增加state，响应react的生命周期。
对于HOC模式来说，优点如下：
- 支持ES6
- 复用性强，HOC为纯函数且返回值为组件，可以多层嵌套
- 支持传入多个参数，增强了适用范围

当然也存在如下缺点：
- 当多个HOC一起使用时，无法直接判断子组件的props是哪个HOC负责传递的
- 多个组件嵌套，容易产生同样名称的props
- HOC可能会产生许多无用的组件，加深了组件的层级

Render Props模式的出现主要是为了解决HOC所出现的问题。优点如下所示：
- 支持ES6
- 不用担心props命名问题，在render函数中只取需要的state
- 不会产生无用的组件加深层级
- render props模式的构建都是动态的，所有的改变都在render中触发，可以更好的利用组件内的生命周期。

当然笔者认为，对于Render Props与HOC两者的选择，应该根据不同的场景进行选择。Render Props模式比HOC更直观也更利于调试，而HOC可传入多个参数，能减少不少的代码量。

Render Props对于只读操作非常适用，如跟踪屏幕上的滚动位置或鼠标位置。 HOC倾向于更好地执行更复杂的操作，例如以上的localStorage功能。

### 参考文献
[Understanding React Render Props by Example](https://levelup.gitconnected.com/understanding-react-render-props-by-example-71f2162fd0f2)

[Understanding React Higher-Order Components by Example](https://levelup.gitconnected.com/understanding-react-higher-order-components-by-example-95e8c47c8006)

[Ultimate React Component Patterns with Typescript 2.8](https://levelup.gitconnected.com/ultimate-react-component-patterns-with-typescript-2-8-82990c516935)
