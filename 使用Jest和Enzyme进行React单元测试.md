单元测试（Unit Testing）又称为模块测试，是对程序模块（也就是函数）进行正确性校验的测试工作。
举个简单的例子，对绝对值函数`abs()`，我们可以编写以下几个测试用例：

1. 输入正数，返回值与输入相同；
2. 输入负数，返回值是输入的相反数；
3. 输入0，返回0；
4. 输入非数值类型，抛出 TypeError。

把这些测试用例放在一个测试模块中，就是一个完整的单元测试。下面用单元测试框架 [mocha](https://mochajs.org/) 和断言库 [chai](http://chaijs.com/) 完成这个单元测试：
```javascript
describe('test abs()', () => {
  it('positive -> positive', () => {
    expect(abs(1)).to.equal(1)
    expect(abs(1.1)).to.equal(1.1)
  })
  it('negative -> positive', () => {
    expect(abs(-1)).to.equal(1)
    expect(abs(-1.1)).to.equal(1.1)
  })
  it('0 -> 0', () => {
    expect(abs(0)).to.equal(0)
  })
  it('throws TypeError', () => {
    expect(function () {
      abs({})
    }).to.throw(TypeError)
  })
})
```
其中，`describe` 和 `it` 是 mocha 中的方法，`describe` 用于定义一个测试模块；`it` 用于定义一条测试用例。`expect` 是 chai 中的方法，用于执行一条断言。
如果单元测试通过了，就说明我们测试的这个函数能够满足所提测试用例的要求。这里要注意的是，通过单元测试只能保证程序在所提供的测试用例范围内是正常工作的，所以测试用例的设计非常重要，应该要覆盖常用的输入组合、边界条件和异常。
那通过单元测试有什么意义呢？

# 1. 单元测试的好处
单元测试的目标是隔离程序部件并证明这些单个部件是正确的。它可以：

1. 允许开发者在未来重构代码，并且确保模块依然正确工作；
2. 提供系统的一种文档记录。通过查看单元测试提供的功能和单元测试中如何使用程序单元，开发者可以直观的理解程序单元的基础API；
3. 一个容易被测试的模块一定是低耦合的，单元测试可以反过来促使开发人员写出更好的代码。为了更方便的进行单元测试，代码中应该避免以下情况：
    - 存在太多条件逻辑
    - 构造函数中做的事情太多
    - 存在太多全局状态
    - 混杂了太多无关的逻辑
    - 存在太多静态方法
    - 存在过多外部依赖

当然单元测试也有一定的局限性：

# 2. 单元测试的局限
1. 单元测试只测试程序单元自身的功能。因此，它不会发现集成错误、性能问题或产品功能层面的问题；
2. 编写测试代码的过程是非常枯燥乏味的。

# 3. 编写单元测试的时机
那么，应该在什么时候编写单元测试呢？[这篇文章](http://www.infoq.com/cn/news/2017/02/writing-good-unit-tests)给出了下面两种选择：
1. 在生产代码完成之后编写单元测试，这称为“测试延后”。这种方式需要在写程序的时候时刻考虑代码的可测试性，并且提交代码时需要经过代码 review 以确保它是可测试的。
2. “测试优先”。这需要在开发之前分析问题，然后编写单元测试，最后编写代码T通过测试。采用这种方法可以获得更高的覆盖率。此外，在编写单元测试的过程中，会更深入的了解问题，对需求容易有更深的把握。这里的测试优先和所谓的测试驱动（TDD）还不是一回事，相关的讨论可以看[这篇文章](https://coolshell.cn/articles/3649.html)。

# 4. 使用 Jest + Enzyme 对 React 项目进行单元测试
首先介绍我们需要用到的工具：
- [Jest](https://facebook.github.io/jest/)
    - 这是 Facebook 出品的一款测试框架。相比 Karma, Mocha 之类测试框架，Jest 的优势有：
        1. 内置了常用的测试工具，比如断言、测试覆盖率等，实现了开箱即用；
        2. 具有快照测试功能，可以简单高效的确保UI不会意外的发生改变；
        3. Jest的测试用例是并行执行的，而且只执行了发生改变的文件所对应的测试，提升了测试速度。
- [Enzyme](http://airbnb.io/enzyme/)
    - Enzyme 是 Airbnb 在 React 官方测试工具库的基础上封装而成的第三方测试库，它模拟了 jQuery 的 API，非常直观，易于使用和学习。
- [redux-mock-store](https://github.com/arnaudbenard/redux-mock-store)
    - 顾名思义，用于模拟 redux 中的 store。
- [nock](https://github.com/node-nock/nock)
    - 用于模拟 HTTP 请求的发出和响应。
- [Sinon](http://sinonjs.org/)
    - Sinon通过创建“测试替身”，将代码中难以被测试的部分替换为更容易被测试的内容，可以用于测试一个函数是否被正确的调用。
限于篇幅，这里不对使用方法具体展开，详情请根据链接查询相关文档。下面结合具体实例谈谈如何对 React 应用进行单元测试。

## 4.1 环境准备
### 4.1.1 安装相关依赖：
```bash
yarn install jest enzyme nock redux-mock-store sinon --dev
```

### 4.1.2 在 package.json 中对 Jest 进行配置：

```shell
{
    "jest":
        "modulePaths": [
          "<rootDir>/src"
        ],
        "moduleNameMapper": {
          "\\.s?css$": "<rootDir>/__test__/scssStub.js"
        },
        "setupFiles": [
          "<rootDir>/__test__/setup.js"
        ]
    }
}
```

其中，`<rootDir>` 是 package.json 文件所在目录。`modulePaths` 指定的是测试代码中 `import` 的根目录；`moduleNameMapper` 是将代码中的 scss/css 文件映射为 js 文件，因为测试不会覆盖样式，也不会解析 scss/css 文件。`scssStub.js` 文件可以简单的写成：

```javascript
module.exports = {}
```

`setupFiles` 指定了启动 Jest 之前要执行的操作，在这里，为：

```javascript
import fetch from 'isomorphic-fetch'
import sinon from 'sinon'
import jest from 'jest'

global.fetch = fetch
global.sinon = sinon
global.jest = jest
```

注意因为执行测试的是 node 环境，并不支持 `fetch` API，所以需要导入并替换 `fetch`。

### 4.1.3 在 package.json 中编写测试启动命令：
```shell
{
    "scripts": {
        "test": "jest",
        "test:dev": "jest --watch"
    }
}
```

`jest --watch` 可以监控代码和测试代码的改动，仅对改变的文件重新进行测试

## 4.2 开始测试
和 React 项目的架构类似，对应的单元测试也可以分为：

- 对组件的测试，包括生命周期、组件内部方法等
- 对 actions 的测试
- 对 reducer 的测试

这几大部分。下面依次举例说明。

### 4.2.1 对组件的测试
#### 4.2.1.1 对生命周期的测试

对于

- constructor
- componentWillReceiveProps
- shouldComponentUpdate
- componentWillUpdate
- render
- componentWillUnmount

可以使用 Enzyme 中的 `shallow` 方法加载组件，例如：

```javascript
it('componentWillReceiveProps', () => {
  let component = shallow(<Card cardType={''} />)
  sinon.spy(Card.prototype, 'componentWillReceiveProps')
  component.setProps({
    cardType: 'Person'
  })
  expect(Card.prototype.componentWillReceiveProps.calledOnce).toBeTruthy
})
```

其中，Spy 是 Sinon 提供的特殊函数，它会记录被调用时的参数、this、调用次数， 以及返回值、异常等等，用来测试一个函数是否被正确地调用。在这里用来记录 `componentWillReceiveProps` 方法是否被调用。`shallow` 是 Enzyme 中加载组件的一种方法，它只渲染当前组件，只能能对当前组件做断言；而 `mount` 会渲染当前组件以及所有子组件，对所有子组件也可以做上述操作，此外，`mount` 是一种 'Full DOM rendering' 的方法。
所以，对于

- componentDidMount
- componentDidUpdate

只能用 `mount` 方法进行加载，例如：

```javascript
describe('component lifecyle demo', () => {
  let component
  beforeEach(() => {
    component = mount(<Card cardType={''} />)
  })
  it('componentDidMount', () => {
    sinon.spy(Card.prototype, 'componentDidMount')
    expect(Card.prototype.componentDidMount.calledOnce).toBeTruthy
  })

  it('componentDidUpdate', () => {
    component.setProps({
      cardType: 'Person'
    })
    sinon.spy(Card.prototype, 'componentDidUpdate')
    expect(Card.prototype.componentDidUpdate.calledOnce).toBeTruthy
  })
}
```

### 4.2.2 对于事件的测试
```javascript
it('simulates click events', () => {
  const onButtonClick = sinon.spy()
  const wrapper = mount((
    <Foo onButtonClick={onButtonClick} />
  ))
  wrapper.find('button').simulate('click')
  expect(onButtonClick.calledOnce).toBeTruthy
});
```
这里，将 spy 生成的函数 mock 为事件处理函数，然后用 Enzyme 中的 `simulate` 方法模拟点击事件，然后用断言确认事件处理函数被调用。从 `wrapper.find('button')` 可以看到 Enzyme 遵循了 jQuery 风格。

### 4.2.3 对于组件方法的测试

例如，有如下组件：
```javascript
export class Card extends React.Component {
  constructor (props) {
    super(props)

    this.cardType = 'initCard'
  }

  changeCardType (cardType) {
    this.cardType = cardType
  }
  ...
}
```

可以对 `changeCardType` 方法进行如下测试：
```javascript
it('changeCardType', () => {
  let component = shallow(<Card />)
  expect(component.instance().cardType).toBe('initCard')
  component.instance().changeCardType('testCard')
  expect(component.instance().cardType).toBe('testCard')
})
```
其中，`instance` 方法可以用于获取组件的内部成员对象。

### 4.2.4 使用 snapshot 进行 UI 测试
这是 Enzyme 的一大特色，可以使用如下方式生成 snapshot：

```javascript
import renderer from 'react-test-renderer'

describe('(Component) Card -- snapshot', () => {
  it('capture snapshot of Chart', () => {
    const renderedValue = renderer.create(<Card />).toJSON()
    expect(renderedValue).toMatchSnapshot()
  })
})
```
snapshot 可以测试组件的渲染结果是否符合预期，所谓预期就是指你上一次录入保存的结果，`toMatchSnapshot` 方法会去帮你对比这次将要生成的 DOM 结构与上次的区别。如果 DOM 发生了变化，Jest 将会输出发生差异的 DOM 结构，然后开发者可以选择返回代码进行修改，或者按下 `u` 对 snapshot 进行更新。so easy~

这是一个很重要的功能，否则要对 DOM 结构的更改编写单元测试将是非常费时费力的。对于每个组件，理论上都应该覆盖 snapshot。

### 4.2.5 对 actions 的测试
#### 4.2.5.1 普通的 actions 测试
对 actions 的测试可以分为两种情况，一种就是普通的 action，如下：
```javascript
export function setOperateChartStatus (status) {
  return {
    'type': 'SET_OPERATE_CHART_STATUS',
    status
  }
}
```
可以写出如下测试：
```javascript
import * as actions from 'actions/Chart'

it('setOperateChartStatus', () => {
  const status = actions.setOperateChartStatus('init')
  expect(status).toEqual({ type: 'SET_OPERATE_CHART_STATUS', status: 'init' })
})
```
可以看出这就是一个简单的函数测试，按下不表。

#### 4.2.5.2 异步 actions 测试
对于使用了如 redux-thunk 这种异步中间件的 actions，就有些麻烦了，如：
```javascript
export function fetchTest () {
  function fetchSuccess (body) {
    return {
      type: 'FETCH_DATA_SUCCESS',
      body
    }
  }

  return (dispatch) => {
    return fetch('http://example.com/todos', {
      method: 'post'
    }).then((res) => res.json())
      .then((json) => dispatch(fetchSuccess(json.body)))
  }
}
```

这里使用了 redux-thunk 进行异步处理，要对这个异步 action 进行测试，需要这样写：

```javascript
import thunk from 'redux-thunk'
import configureStore from 'redux-mock-store'
import * as actions from 'actions/Chart'

describe('actions test', () => {
  const middlewares = [thunk]
  const mockStore = configureStore(middlewares)

  afterEach(() => {
    nock.cleanAll()
  })

  it('fetchTest', () => {
    // 模拟请求
    nock('http://example.com/')
      .post('/todos')
      .reply(200, { body: { todos: ['do something'] } })

    const expectedActions = [
      { type: 'FETCH_DATA_SUCCESS', body: { todos: ['do something'] } }
    ]
    const store = mockStore()

    store.dispatch(actions.fetchTest()).then(() => {
      // async actions 的返回值
      expect(store.getActions()).toEqual(expectedActions)
    })
  })
})
```

### 4.2.6 对 reducer 的测试
对 reducer 的测试也相当简单，就是对一个纯函数的测试，直接上例子：
```javascript
export function renderChartStatus (state = { status: '' }, action) {
  switch (action.type) {
    case 'GET_CHART_DATA_FAIL':
      return { status: 'fail', msg: action.msg };
    case 'GET_CARD_DATA_FAIL':
      return { status: 'fail', msg: action.msg };
    case 'SET_RENDER_CHART_STATUS':
      return { status: action.status };
    default:
      return state;
  }
}
```

测试：

```javascript
import { renderChartStatus } from 'reducers/Chart'

describe('(reducers) chart reducers', () => {
  it('renderChartStatus', () => {
    let state = { status: 'haha' }
    state = renderChartStatus(state, { type: 'SET_RENDER_CHART_STATUS', status: 'init' })
    expect(state).toEqual({ status: 'init' })
  })
})
```

# 5. 总结与展望
本文是过去一周在图谱项目中实践单元测试的总结，旨在介绍使用 Jest + Enzyme 进行 React 单元测试的整个流程。对于一些更深入的细节，如测试用例的设计、测试框架的深入使用等尚未涉及，留坑后续再填。

此外，使用 d3 进行绘图的相关组件和方法进行测试时，在测试环境下，遇到了 Enzyme 不支持在 React 项目中对 DOM 的进行操作的问题。虽然 React 确实不推荐直接操作 DOM，Enzyme 的这种限制无可厚非，但是 d3 又必须要操作 DOM 才能进行 SVG 绘制。对于 Enzyme 是否有相关方法解决这一矛盾，亦或是只能采取其他方式进行测试还有待研究。

# 6. 参考文献
1. http://www.aliued.com/?p=4095
2. http://insights.thoughtworks.cn/react-and-unit-testing/
3. https://www.zhihu.com/question/28729261
4. https://zh.wikipedia.org/wiki/%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95
5. http://www.infoq.com/cn/news/2017/02/writing-good-unit-tests
6. http://www.infoq.com/cn/news/2012/02/I-Hate-Unit-Test
7. https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/00140137128705556022982cfd844b38d050add8565dcb9000
8. https://coolshell.cn/articles/8209.html
9. https://75team.com/post/5-questions-every-unit-test-must-answer.html
10. https://75team.com/post/5-questions-every-unit-test-must-answer.html
11. http://www.techug.com/post/19-unit-test-code-tips.html
12. http://nicholas.ren/2012/10/21/my-understanding-on-unit-testing.html
13. https://medium.com/netscape/testing-a-react-redux-app-using-jest-and-enzyme-b349324803a9
14. https://github.com/airbnb/enzyme/issues/799
15. http://zcfy.cc/article/sinon-tutorial-javascript-testing-with-mocks-spies-stubs-422.html
16. http://echizen.github.io/tech/2017/02-12-jest-enzyme-method
