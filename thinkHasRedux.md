### 背景

我一个前端，今年第一份工作就是接手一个 APP 的开发。。。一个线下 BD 人员用的推广 APP，为了让我这个一天原生开发都没有学过的人能快速开发上线，于是乎就选择了 react-native 作为开发框架，既然主框架有了，接下来就是主要的逻辑框架的选择了。

一直以来社区里面关于 react 的状态管理都是推荐 Flux 思想的实践者 Redux ，关于这个 Redux 国内已经有了很多很多的分析和讲解了，自然资料也是最多，坑最少的选择了，所以这次的开发便选择了 Redux 全家桶来作为整个 APP 的状态管理库了。

### 疑问

熟悉 Redux 的同学一定非常熟悉这张图了![](https://facebook.github.io/flux/img/flux-simple-f8-diagram-explained-1300w.png)
这张经典的状态管理流程图清晰明确的表现出了整个 Flux 的运行流程，可以看到所有的状态的修改都是进过了 Dispatcher 事件，能触发 dispatcher 的也只有 action，这样一个流程化的过程可以确保整个 store 的修改有据可查，我们来看看网上都是怎么说 flux 的好处的：

> * 数据状态变得稳定同时行为可预测
> * 所有的数据变更都发生在store里
> * 数据的渲染是自上而下的
> * view层变得很薄，真正的组件化
> * dispatcher是单例的

可以看到，对于 flux，大家对它的评价是非常好的，的确在上面说到的方面 flux 的思路确实做的非常的好，并且让整个状态的改变变得规范了起来，但另外一方面，这却让原本灵活的 JavaScript 变得 “Java” 了起来。对于一个状态的修改我们不得不去写一堆模板代码，任何状态的修改都需要发起一个 action，然后触发一个 dispatcher，最后才能修改到 store，并且熟悉 Redux 的同学肯定写过这样的代码：

```javaScript
// reduces.js
let initialState = {
    userLogin: {
        name: '',
        state: '',
        payState: '',
        isDonor: false
    }
};
function login(state = initialState.userLogin, action) {
    switch (action.type) {
        case GET_CODE:
            state.state = action.fetchData.errors ? 'GET_CODE_ERROR' : 'GET_CODE_OK';
            return Object.assign({}, state, action.fetchData);
        case FETCH_OVER:
            state.payState = null;
            state.state = null;
            return Object.assign({}, state);
        case SUBMIT_LOGIN_FORM:
            state.state = action.fetchData.errors ? 'SUBMIT_LOGIN_ERROR' : 'SUBMIT_LOGIN_OK';
            state.name = action.username;
            return Object.assign({}, state, action.fetchData);
        case ADD_PAY_PASS:
            state.payState = action.fetchData.errors ? 'SUBMIT_LOGIN_ERROR' : ADD_PAY_PASS;
            return Object.assign({}, state, action.fetchData);
        case CHANGE_PWD:
            state.state = action.fetchData.errors ? 'SUBMIT_LOGIN_ERROR' : 'SUBMIT_LOGIN_OK';
            return Object.assign({}, state, action.fetchData);
        default:
            return state
    }
}

// action.js
export function handleSubmit(values) {
    const {name, pass, code} = values;
    return dispatch => {
        webApi.postApi('api/accounts/login', {
            account: name,
            password: pass,
            code: code
        }, (err, data) => {
            let fetchData = {};
            if (data.status === GLOBAL_MSG.reqSucCode) {
                fetchData = data;
            } else {
                fetchData.errors = data.errors[0];
            }
            return dispatch({type: SUBMIT_LOGIN_FORM, fetchData, username: name});
        });
    }
}

// view/login.js
handleSubmit(e) {
    e.preventDefault();
    const {form, handleSubmit} = this.props;
    form.validateFields((err, values) => {
        if (!err) {
            handleSubmit(values);
        }
    });
}
```
上面的 demo 代码只是一个片段，它是一个简单的登录功能，可以看到，严格的规范导致我们需要为了修改动一个状态就要调用起码三个文件里面的内容，如果你的项目很大，或者状态修改分的很细，那么你面对的就不是三个文件了，可能会是更多的文件，当然，你也可以把它们全都写在一起。。如果你的同事不砍你的话。。。

还有一个问题，那就是异步修改的问题，这个异步发起的状态是在 action 里面呢还是在 dispatcher 里面呢？那么就真的没有更好的解决办法了嘛？哪怕是语法糖也好呀。

所以，现在在 github 上面出现了很多的“最佳实践”，每个团队都拿出了自己的解决办法，于是我们有了很多现成的解决方案，最后我们选择了阿里家的 [D.Va](https://github.com/dvajs/dva) 来解决我们工作中遇到的上述问题，

### 解决

下面我们就来聊聊这个“最佳实践”--Dva，来看看 dva 的 demo 是怎么写的吧：
```javaScript
import React from 'react';
import dva, { connect } from 'dva';
import { Route } from 'dva/router';

// 1. Initialize
const app = dva();

// 2. Model
app.model({
  namespace: 'count',
  state: 0,
  reducers: {
    ['count/add'  ](count) { return count + 1 },
    ['count/minus'](count) { return count - 1 },
  },
});

// 3. View
const App = connect(({ count }) => ({
  count
}))(function(props) {
  return (
    <div>
      <h2>{ props.count }</h2>
      <button key="add" onClick={() => { props.dispatch({type: 'count/add'})}}>+</button>
      <button key="minus" onClick={() => { props.dispatch({type: 'count/minus'})}}>-</button>
    </div>
  );
});

// 4. Router
app.router(<Route path="/" component={App} />);

// 5. Start
app.start(document.getElementById('root'));
```
我们可以看到，在上述的例子中精简了不少“啰嗦”的代码，在正真的业务中我们其实只需要关系 view 文件和 model 文件这两个中的内容就可以了。
> 5 步 4 个接口完成单页应用的编码，不需要配 middleware，不需要初始化 saga runner，不需要 fork, watch saga，不需要创建 store，不需要写 createStore，然后和 Provider 绑定，等等。但却能拥有 redux + redux-saga + ... 的所有功能。

这是 demo 下的一段话，确实，这对于我们的项目组来说是非常非常有帮助的了，减少了不少的工作量，在我开发 react-native 应用的时候，整个的开发过程变得轻松了很多，我不需要写那么多代码了，有了参考我可以很无脑的堆砌业务代码了，而且不太需要关心逻辑的性能等等各种各样的琐事。这似乎完美的解决了我遇到的所有问题。。。

### 反思

可是真的是这样的嘛？我真的需要 Redux 嘛？我究竟是需要它什么好处呢？如果我自己管理 stroe 就不可以嘛？当然是可以的了，而且最早我就是这么做的，那么我们再回过头来看看 Flux 所带来的那几个好处。

首先是状态可回溯，因为函数式编程中提到的副作用的原因，每次对 store 的修改都应该是旧 store 的一个拷贝，两个 store 是独立的，这样就可以像快照一样的保存住每个时间段 store 的状态来了，这样如果我们需要回滚到之前某一个状态就会变得非常的容易，那么，这个时候就出现了一个面试时的一个考点了，深拷贝和浅拷贝的问题，很显然，新人在对 store 做修改的时候应该都会写过这样的代码吧：
```javaScript
var newStore = OldStore.xxx;
```
如果你写过这样的代码，那么你会发现你依旧无法实现状态的回溯，因为旧的 store 还是被改变了，那么有的小伙伴就说了，我可以用深拷贝呀，但是深拷贝需要在堆中重新分配内存，并且把源对象所有属性都进行新建拷贝，才能保证拷贝后的对象与原来的对象完全隔离，互不影响。这样做带来的一个问题就是内存的增加，可以大胆的想象一下如果你的应用状态非常的多，或者做了非常多的操作之后，内存会变成什么样。

这个时候你又说了，那我可以用 immutable.js 啊，没错~Redux 全家桶又迎来了一位新的成员，于是你又四处去找immutable.js的资料回来看，这么不讨论immutable.js的重要性，在对于不可变数据的处理上它的确是一个不错的选择，但反过来思考一下，我真的需要不可变数据嘛？数据可变又会怎么样呢？这个时候你又要说了：废话，没有不可变数据我怎么回溯状态！

每个业务都有自己的需求点，放在我开发的项目上，其实是不需要这个功能的，那么换句话说，在我这里，我并不需要这个功能，状态回溯功能我个人认为是属于一个锦上添花的功能，而不是一个刚需，当然如果你的业务需要这个特性，就当我没说~

接下来我们聊聊状态的改变，flux 让我们通过 action 去触发 dispatch 来修改状态，这对于那些复杂巨大的项目来说是非常有必要的，因为那么多文件，那么多状态，单靠人力去维护是非常困难的，但是这里又有个问题了，究竟多大的项目叫大项目呢？上万行的代码？还是上百个文件？你的项目真的达到了这个程度了嘛？如果是，那么再想想如果优化一下代码逻辑，项目结构，还会有这么大的代码量吗？在这里说个不太恰当的例子，我刚参加工作的时候一个功能我能写好几百行，然后到了 code review 的时候那些老道的同事总能用几十行的代码来完成和我一样的功能，甚至比我的还要好，那你说如果一个中小型项目全都是我这种质量的代码去堆砌的话，它是不是就变成了一个大型项目呢？

显然不是这样的，一个项目的大小不单靠代码的量去衡量，这里面有很多方面的影响，依赖繁多、功能繁琐、逻辑复杂。这些都是大项目的特点，如果你是 bat 大厂的同学，那应该也不会看我这个了，而对于和我一样的小厂开发来说，我们手里的项目真的是一个巨无霸嘛？我想快速迭代的需求恐怕才是第一要解决的问题了，如果代码书写规范，逻辑实现优雅，结构组织清晰，很大程度上你手里的项目是一个小而美的项目，在这种情况下，为什么不能换个思路呢？

前段时间我看到了 mobx.js，一个新的状态管理库，但思路却完全不一样，它没有不可变数据的要求，它代表的是另外一种开发思路，使用 Mobx，代码会更加清晰，也便于维护，当然前提是你的项目是一个中小型项目，其实用过 vue 的同学去看它，就会觉得相当的亲切，对于 mobx 的资料现在网上也有很多了，我在这里就不做过多的介绍了，前段时间我抱着试试看的心态，尝试着重构了app 中的部分代码，相较于 dva 模式下的代码，确实精简了不少，也由此看来我的项目中充斥着大量的“冗余”代码，从而看起来非常的庞大，通过精简逻辑，优化结构，采用 mobx 反而更“优雅”了。


### 总结

虽然最后一次的重构并没有上线我就离职了，但是也是这一次的重构让我认识到了大家说的“最佳实践”并不一定就是我项目的“最佳实践”，很多时候还是要考虑自身的局限性。面对前端丰富的解决方案，如何选择时候自己的才是我们这些开发者需要考虑的，不同的场景有不同的语言，同样不同的业务也有不同的框架，曾经我以为可以 redux 全家桶一桶走天下的，但是现在想想还是“too young too simple sometimes native”啊。现在看看那些招聘或者求职上清一色写的 Redux，不妨仔细想想，真的需要吗？
