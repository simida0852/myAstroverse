---
title: 如何在React中使用Props
slug: 如何在React中使用Props
pubDate: 2022-05-30
tags: 
  - 前端
  - React
  - 教程
description: 介绍React Props的概念以及如何在React中使用Props
author: xf
cover: src/images/nestjs-1.webp
coverAlt: react
category:
  - 前端
---

- [React Props 组件的示例](#react-props-组件的示例)
- [React Props vs State](#react-props-vs-state)
- [如何将Props从子组件传递给父组件](#如何将props从子组件传递给父组件)
- [React Props 只是一个沟通的渠道](#react-props-只是一个沟通的渠道)
- [React Props的解构](#react-props的解构)
- [React Props 扩展操作符](#react-props-扩展操作符)
- [React Props rest解构操作符](#react-props-rest解构操作符)
- [具有默认值的React Props](#具有默认值的react-props)
- [React中的children属性](#react中的children属性)
- [如何通过Props传递组件](#如何通过props传递组件)
- [Children 作为一种函数](#children-作为一种函数)

对于每个刚接触React的开发人员来说都会或多或少的对 `React Props` 感到困惑，因为它们在其他框架中从未被提及，也很少有对它的相关解释。它们是开发人员在掌握了React的JSX语法后，在React中早期要学习的东西之一。从本质上说，React组件Props是用来在组件之间传递数据的。在本教程中，我想通过React Props的例子，一步一步地解释React中的Props。

<!-- more -->

## React Props 组件的示例

通常在学习 React 时，你会使用 React 的 JSX 语法来向浏览器呈现一些东西。基本上 JSX 把 HTML 和 javaScript 混合在一起，以达到两者兼得。

```javascript
import * as React from 'react';

const App = () => {
  const greeting = 'Welcome to React';

  return (
    <div>
      <h1>{greeting}</h1>
    </div>
  );
}

export default App;
```

接下来，我们将实现第一个 React 函数组件:

```javascript
import * as React from 'react';

const App = () => {
  return (
    <div>
      <Welcome />
    </div>
  );
};

const Welcome = () => {
  const greeting = 'Welcome to React';

  return <h1>{greeting}</h1>;
};

export default App;
```

这次重构后的一个常见问题：如何将数据从一个React组件传递到另一个组件？毕竟，新组件应该渲染一个动态的问候语，而不是在新组件中定义的静态问候语。它的行为应该像一个函数，应为我这里需要传递参数。

然后我们来修改React的Props,达到React中可以将数据从一个组件传递到另一个组件的目的。通过自定义HTML属性，用JSX的语法将数据赋给这些属性。

```javascript
import * as React from 'react';

const App = () => {
  const greeting = 'Welcome to React';

  return (
    <div>
      <Welcome text={greeting} />
    </div>
  );
};

const Welcome = (Props) => {
  return <h1>{Props.text}</h1>;
};

export default App;
```

由于在函数组件中的第一个参数始终是Props(它只是一个javaScript对象，保存了从一个组件传递到另一个组件的所有数据)，因此可以尽早解构Props。这被称之为React Props的解构

```javascript
import * as React from 'react';

const App = () => {
  const greeting = 'Welcome to React';

  return (
    <div>
      <Welcome text={greeting} />
    </div>
  );
};

const Welcome = ({ text }) => {
  return <h1>{text}</h1>;
};

export default App;
```

正如您所看到的，Props使您能够沿着组件树将值从一个组件传递到另一个组件。在前面的示例中，它只是一个字符串变量。但Props可以是任何javaScript数据类型，从整数到数组。通过Props，你甚至可以传递React组件，这将在后面介绍。

值得一提的是，你也可以直接定义 Props 而不用声明变量:

```javascript
import * as React from 'react';

const App = () => {
  return (
    <div>
      <Welcome text={"Welcome to React"} />
    </div>
  );
};

const Welcome = ({ text }) => {
  return <h1>{text}</h1>;
};

export default App;
```

在javaScript字符串的情况下，您也可以将其作为Props传入双引号(或单引号)中

```javascript
import * as React from 'react';

const App = () => {
  return (
    <div>
      <Welcome text="Welcome to React" />
    </div>
  );
};

const Welcome = ({ text }) => {
  return <h1>{text}</h1>;
};

export default App;
```

但是您也可以使用这些内联Props传递其他的 javaScript 数据结构。对于 React 初学者来说，javaScript 对象可能会令人困惑。因为你有两个花括号: 一个用于 JSX，另一个用于 JSON:

```javascript
import * as React from 'react';

const App = () => {
  return (
    <div>
      <Welcome text={{ greeting: 'Welcome to React' }} />
    </div>
  );
};

const Welcome = ({ text }) => {
  return <h1>{text.greeting}</h1>;
};

export default App;
```

当声明数据为一个合适的 javaScript 对象时，它变得更具可读性:

```javascript
import * as React from 'react';

const App = () => {
  const greetingObject = { greeting: 'Welcome to React' };

  return (
    <div>
      <Welcome text={greetingObject} />
    </div>
  );
};

const Welcome = ({ text }) => {
  return <h1>{text.greeting}</h1>;
};

export default App;
```

大多数React初学者在第一次向React中的本地HTML元素的样式属性传递一个样式对象时注意到了这一点。

```javascript
import * as React from 'react';

const App = () => {
  return (
    <div>
      <Welcome text={{ greeting: 'Welcome to React' }} />
    </div>
  );
};

const Welcome = ({ text }) => {
  return <h1 style={{ color: 'red' }}>{text.greeting}</h1>;
};

export default App;
```

基本上这就是在 React 中Props如何从一个组件传递到另一个组件。正如您可能已经注意到的，Props只是在 React 应用程序的组件层次结构中从顶部到底部传递。没有办法将Props从子组件传递到父组件。我们将在本教程的后面重新考虑这个警告。

需要注意的是 React 的Props是只读的(不可变的)。作为一名开发人员，您永远不应该变更Props，而应该只在组件中读取它们。不过，您可以从它们中派生新的值(请参阅后面的计算属性)。毕竟，Props只用于将数据从父组件传递给子组件 React。本质上，Props只是将数据传输到组件树的工具。

## React Props vs State

在React中将Props从一个组件传递到另一个组件并不会使组件交互，因为Props是只读的，因此也是不可变的。如果您想要交互式React组件，则必须使用React State引入有状态值。通常情况下，状态通过React的useState钩子与React组件共存

```javascript
import * as React from 'react';

const App = () => {
  const greeting = 'Welcome to React';

  const [isShow, setShow] = React.useState(true);

  const handleToggle = () => {
    setShow(!isShow);
  };

  return (
    <div>
      <button onClick={handleToggle} type="button">
        Toggle
      </button>

      {isShow ? <Welcome text={greeting} /> : null}
    </div>
  );
};

const Welcome = ({ text }) => {
  return <h1>{text}</h1>;
};

export default App;
```

在上一个例子中，App组件使用一个叫做isShow的状态值和一个状态更新函数来更新事件处理程序中的这个状态。根据isShow的布尔状态，父组件通过使用条件性渲染来渲染或不渲染其子组件。

这个例子显示了状态与Props的不同。Props只是一个在组件树上传递信息的工具，而状态可以随着时间的推移而改变，以创建交互式用户界面。下一个例子展示了当状态被传递给子组件时，它如何成为Props。尽管状态在子组件中成为Props，但它仍然可以在父组件中通过状态更新函数作为状态被修改。一旦被修改，状态就会作为 "修改过的 "Props被传递下去。

```javascript
import * as React from 'react';

const App = () => {
  const [greeting, setGreeting] = React.useState('Welcome to React');
  const [isShow, setShow] = React.useState(true);

  const handleToggle = () => {
    setShow(!isShow);
  };

  const handleChange = (event) => {
    setGreeting(event.target.value);
  };

  return (
    <div>
      <button onClick={handleToggle} type="button">
        Toggle
      </button>

      <input type="text" value={greeting} onChange={handleChange} />

      {isShow ? <Welcome text={greeting} /> : null}
    </div>
  );
};

const Welcome = ({ text }) => {
  return <h1>{text}</h1>;
};

export default App;
```

换句话说，我们可以说载体（Props）中的值（state）已经改变。子组件并不关心Props里面的值是否是有状态的值--它只是把它们看作来自父组件的Props。由于一个组件（这里是父组件）的每一个状态变化都会导致这个组件和所有子组件的重新渲染，所以子组件最后只是接收更新的Props。

总之，每次状态改变，受影响的组件及其所有子组件的渲染机制都会被触发。这就是整个组件树成为交互式的原因，因为毕竟有状态的值（state）是作为Props传递给子组件的，一旦一个组件中的状态发生变化，可能作为Props传递给子组件，所有重新渲染的子组件都会使用新的Props。

## 如何将Props从子组件传递给父组件

当Props只能从父组件传递给子组件时，子组件如何与父组件进行交流？这是React初学者在了解了React中的Props后的一个常见问题，答案很简单：没有办法从子组件向父组件传递Props。

让我们重温一下之前的例子，但这次是用一个新的可重用组件Button来实现之前实现的显示/隐藏切换功能。

```javascript
import * as React from 'react';

const App = () => {
  const [greeting, setGreeting] = React.useState('Welcome to React');

  const handleChange = (event) => {
    setGreeting(event.target.value);
  };

  return (
    <div>
      <Button label="Toggle" />

      <input type="text" value={greeting} onChange={handleChange} />

      {isShow ? <Welcome text={greeting} /> : null}
    </div>
  );
};

const Button = ({ label }) => {
  const [isShow, setShow] = React.useState(true);

  const handleToggle = () => {
    setShow(!isShow);
  };

  return (
    <button onClick={handleToggle} type="button">
      {label}
    </button>
  );
};

const Welcome = ({ text }) => {
  return <h1>{text}</h1>;
};

export default App;
```

到目前为止，新的Button组件管理着它自己的同位状态。由于Button组件管理着isShow的状态值，所以没有办法把它作为Props传递给父组件，而在父组件中，欢迎组件的条件渲染需要它。因为我们无法访问App组件中的isShow值，所以应用程序会中断。为了解决这个问题，让我们进入如何在React中提升状态。

```javascript
import * as React from 'react';

const App = () => {
  const [greeting, setGreeting] = React.useState('Welcome to React');
  const [isShow, setShow] = React.useState(true);

  const handleChange = (event) => {
    setGreeting(event.target.value);
  };

  const handleToggle = () => {
    setShow(!isShow);
  };

  return (
    <div>
      <Button label="Toggle" onClick={handleToggle} />

      <input type="text" value={greeting} onChange={handleChange} />

      {isShow ? <Welcome text={greeting} /> : null}
    </div>
  );
};

const Button = ({ label, onClick }) => {
  return (
    <button onClick={onClick} type="button">
      {label}
    </button>
  );
};

const Welcome = ({ text }) => {
  return <h1>{text}</h1>;
};

export default App;
```

应用程序再次工作。重要的成分是：App组件在Props中向Button组件传递了一个函数。这个函数在React中被命名为回调处理程序（因为它通过Props从组件传递到组件，并回调到原点组件），被用于Button组件的点击处理程序。

虽然Button不知道这个函数的业务逻辑，但它必须在按钮被点击的时候触发这个函数。在上面的App组件中，当传递的函数被调用时，状态被改变，因此父组件和所有的子组件都会重新渲染。
如前所述，没有办法从子组件向父组件传递Props。但是你总是可以把函数从父组件传给子组件，而子组件则利用这些函数，这些函数可能会改变上面一个父组件的状态。一旦状态改变了，状态就会再次作为Props向下传递。所有受影响的组件将再次渲染。

## React Props 只是一个沟通的渠道

接收Props的组件不知道数据的来源。它只在React中看到一个名为Props的javaScript对象。其中:Props可以来自父组件，也可以来自组件层次之上的任何地方。并且数据可以是有状态的或者无状态的。

例如，Props不仅可以从父组件传递到子组件，还可以从祖先组件传递到后代组件。

```javascript
import * as React from 'react';
const App = () => {
  const greeting = {
    title: 'React',
    description: 'Your component library for ...',
  };
  return (
    <div>
      <Welcome text={greeting} />
    </div>
  );
};

const Welcome = ({ text }) => {
  return (
    <div>
      <Headline title={`Welcome to ${text.title}`} />
      <Description paragraph={text.description} />
    </div>
  );
};

const Headline = ({ title }) => <h1>{title}</h1>;

const Description = ({ paragraph }) => <p>{paragraph}</p>;

export default App;
```

如上面代码所示，Headline组件和Description组件都不知道信息是来自Welcome组件还是 App 组件。如果通过使用 React 的 useState Hook，greeting 在 App 组件中成为一个有状态的值，情况也是如此。然后，有状态greeting仅仅是文本，仅是Welcome 组件的 Props 中的一个属性。它最终将其传递给子组件。

最后，要强调的一点是，仔细查看上面示例中的 Welcome 组件。它将 title prop 传递给 Headline 组件，但不仅仅使用 text.title，而是使用它创建一个新字符串。在不修改 Props 的情况下，组件使用 title 属性从中派生一个新值。这个原理在 React 中被称为计算属性。

## React Props的解构

之前，我们已经简要地学习了React中的Props解构，并且在前面的所有Props示例中都使用了它。让我们快速回顾一下。React 中的 Props基本上是从父组件传递给子组件的所有数据。在子组件中，Props可以作为参数在函数中访问:

```javascript
import * as React from 'react';
const App = () => {
  return (
    <div>
      <Welcome text="Welcome to React" />
    </div>
  );
};
const Welcome = (Props) => {
  return <h1>{Props.text}</h1>;
};
```

如果我们将Props理解为从父组件传达到子组件的载体，我们通常是不会直接使用这个载体，而是只想使用其中的内容。因此，我们可以分解传入的参数

```javascript
import * as React from 'react';
const App = () => {
  return (
    <div>
      <Welcome text="Welcome to React" />
    </div>
  );
};
const Welcome = (Props) => {
  const { text } = Props;
  return <h1>{text}</h1>;
};
```

同时我们也可以在函数中分解javaScript对象，所以我们可以省略中间变量赋值

```javascript
import * as React from 'react';
const App = () => {
  return (
    <div>
      <Welcome text="Welcome to React" />
    </div>
  );
};
const Welcome = ({ text }) => {
  return <h1>{text}</h1>;
};
```

如果将多个Props传递给一个子组件，我们可以将它们全部拆分:

```javascript
import * as React from 'react';
const App = () => {
  return (
    <div>
      <Welcome text="Welcome to React" myColor="red" />
    </div>
  );
};
const Welcome = ({ text, myColor }) => {
  return <h1 style={{ color: myColor }}>{text}</h1>;
};
```

然而，有时候我们实际上把Props作为对象，所以让我们在接下来的章节中讨论它们。

## React Props 扩展操作符

将对象的所有属性传递给子组件的策略是使用了[javaScript扩展操作符](https://developer.mozilla.org/en-US/docs/Web/javaScript/Reference/Operators/Spread_syntax)。 扩展操作符在React中是一个非常有用的强大功能，你可以看到人们把它称为 **React ... Props** 语法，即使它不是真正的 React 功能，而只是来自 javaScript 的一种语法。

```javascript
import * as React from 'react';
const App = () => {
  const greeting = {
    title: 'React',
    description: 'Your component library for ...',
  };
  return (
    <div>
      <Welcome {...greeting} />
    </div>
  );
};
const Welcome = ({ title, description }) => {
  return (
    <div>
      <Headline title={`Welcome to ${title}`} />
      <Description paragraph={description} />
    </div>
  );
};
const Headline = ({ title }) => <h1>{title}</h1>;

const Description = ({ paragraph }) => <p>{paragraph}</p>;

export default App;
```

Props扩展可用于将具有键值对的整个对象扩展到子组件。它与将对象属性的每个属性逐个传递给组件的效果相同。例如，有时你有一个组件，它不关心Props，只是把它们传递给下一个组件

```javascript
import * as React from 'react';
const App = () => {
  const title = 'React';
  const description = 'Your component library for ...';
  return (
    <div>
      <Welcome title={title} description={description} />
    </div>
  );
};
const Welcome = (Props) => {
  return (
    <div style={{
      border: '1px solid black',
      height: '200px',
      width: '400px',
    }}>
      <Message {...Props} />
    </div>
  );
};
const Message = ({ title, description }) => {
  return (
    <>
      <h1>{title}</h1>
      <p>{description}</p>
    </>
  );
}
export default App;
```

请注意，扩展属性的键值值对也可以被覆盖:

```javascript
const Welcome = (Props) => {
  return (
    <div>
      <Message {...Props} title="javaScript" />
    </div>
  );
};

```

如果Props的扩张属性放在最后，那么所有之前的属性如果出现在Props中就会被覆盖:

```javascript
const Welcome = (Props) => {
  return (
    <div>
      <Message title="javaScript" {...Props} />
    </div>
  );
};
// Message prints title "React"
```

毕竟，可以始终使用 spread 操作符将 javaScript 对象的每个键/值对方便地分配给 HTML 元素的属性。

## React Props rest解构操作符

javaScript的[rest解构操作符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)也可以应用于React Props。让我们看一个Props rest解构操作符的例子。首先，我们定义一个带有内联处理程序的按钮，该处理程序增加数字的状态。按钮已经提取为可重用组件

```javascript
import * as React from 'react';
const App = () => {
  const [count, setCount] = React.useState(0);
  return (
    <div>
      <Button label={count} onClick={() => setCount(count + 1)} />
    </div>
  );
};
const Button = ({ label, onClick }) => (
  <button onClick={onClick}>{label}</button>
);
export default App;
```

一个HTML按钮可以接收很多属性。例如，在某些情况下禁用按钮是经常出现的情况。所以让我们用 disabled 这个新Props来控制Button组件是否可被点击

```javascript
import * as React from 'react';
const App = () => {
  const [count, setCount] = React.useState(0);
  return (
    <div>
      <Button
        label={count}
        disabled={true}
        onClick={() => setCount(count + 1)}
      />
    </div>
  );
};
const Button = ({ label, disabled, onClick }) => (
  <button disabled={disabled} onClick={onClick}>
    {label}
  </button>
);
export default App;
```

随着时间的推移，我们想要传递给按钮的Props会越来越多，因此按钮组件的函数签名会越来越大。我们可以继续这样做，明确Button组件接收到的每一个Props。但是，也可以使用JavaScript的rest解构，它从一个没有被解构的对象中收集所有剩余的属性

```javascript
const Button = ({ label, onClick, ...others }) => (
  <button disabled={others.disabled} onClick={onClick}>
    {label}
  </button>
);
```

为了使 Button 组件的实现更加方便，我们可以使用 JavaScript 的 spread 操作符将其余的 Props 扩展到按钮 HTML 元素。通过这种方式，只要我们给 Button 组件传递一个新的 prop，但是没有显式地对它进行重构，它就会自动分配给 Button HTML 元素:

```javascript
const Button = ({ label, onClick, ...others }) => (
  <button onClick={onClick} {...others}>
    {label}
  </button>
);
```

下面的例子展示了如何将一个布尔值作为 true 的内联值传递给一个简写，因为在子组件中这个属性被赋值为 true

总而言之,通过 Props扩展操作符和 rest解构(剩余解构)Props 可以极大的帮助我们为实现细节保持可读性

## 具有默认值的React Props

在某些情况下，您可能希望将默认值作为Props传递。从历史上看，处理它的最佳方法是使用JavaScript的逻辑or操作符。

```javascript
const Welcome = ({ title, description }) => {
  title = title || 'Earth';
  return (
    <div>
      <Title title={`Welcome to ${title}`} />
      <Description description={description} />
    </div>
  );
};
```

你也可以把它作为Props

```javascript
const Welcome = ({ title, description }) => (
  <div>
    <Title title={`Welcome to ${title || 'Earth'}`} />
    <Description description={description} />
  </div>
);
```

令人高兴的是，在现代JavaScript中，在使用解构时可以使用默认值

```javascript
const Welcome = ({ title = 'Earth', description }) => (
  <div>
    <Title title={`Welcome to ${title}`} />
    <Description description={description} />
  </div>
);
```

## React中的children属性

React中的children属性可以用来将React组件组合成其他组件。由于这个特性，您可以在开始元素和结束元素的标记之间放置JavaScript元素或JSX

```javascript
import * as React from 'react';
const App = () => {
  const [count, setCount] = React.useState(0);
  return (
    <div>
      <Button onClick={() => setCount(count + 1)}>
        {count}
      </Button>
    </div>
  );
};
const Button = ({ onClick, children }) => (
  <button onClick={onClick}>{children}</button>
);
export default App;
```

在这种情况下，只有一个字符串放在元素的标签之间。然后在子组件中，你可以使用React的子Props来利用标签之间的所有东西。例如，你可以像本例中那样渲染子Props的内容。在接下来的小节中，您将看到如何将children prop也用作一个函数。

## 如何通过Props传递组件

在你学习 React 的 children prop 之前，它允许你把 HTML/React 元素作为Props传递给组件:

```javascript
const User = ({ user }) => (
  <Profile user={user}>
    <AvatarRound user={user} />
  </Profile>
);
const Profile = ({ user, children }) => (
  <div className="profile">
    <div>{children}</div>
    <div>
      <p>{user.name}</p>
    </div>
  </div>
);
const AvatarRound = ({ user }) => (
  <img className="round" alt="avatar" src={user.avatarUrl} />
);
```

但是，如果您想传递多个 React 元素并将它们放置在不同的位置，该怎么办？不过话又说回来，你不需要使用children prop，因为你只有一个Props，而只需使用普通Props:

```javascript
const User = ({ user }) => (
  <Profile
    user={user}
    avatar={<AvatarRound user={user} />}
    biography={<BiographyFat user={user} />}
  />
);
const Profile = ({ user, avatar, biography }) => (
  <div className="profile">
    <div>{avatar}</div>
    <div>
      <p>{user.name}</p>
      {biography}
    </div>
  </div>
);
const AvatarRound = ({ user }) => (
  <img className="round" alt="avatar" src={user.avatarUrl} />
);
const BiographyFat = ({ user }) => (
  <p className="fat">{user.biography}</p>
);
```

通常这种方法是用于周围的布局组件，以多个组件作为prop的内容。现在你可以动态地将 Avatar 或者 Biography 组件与其他组件进行交换，比如:

```javascript
const AvatarSquare = ({ user }) => (
  <img className="square" alt="avatar" src={user.avatarUrl} />
);
const BiographyItalic = ({ user }) => (
  <p className="italic">{user.biography}</p>
);
```

许多人在 React 中将其称为插槽模式。你可以在 [GitHub](https://github.com/the-road-to-learn-React/React-slot-pattern-example) 上找到这个小项目。这就是《React》中的构图如何闪耀。您不需要改动 Profile 组件。此外，您不需要传递prop，在这种情况下，用户在组件树的下面传递多个级别，而是将其传递给开槽组件

## Children 作为一种函数

Children作为函数或Children作为函数的概念的时候，也被认为是一种Props渲染方式，是React中的高级模式之一(仅次于高阶组件)。实现这个模式的组件可以被称为prop渲染组件。

首先，让我们从渲染prop开始。基本上它是一个作为prop传递的函数。该函数接收参数(在本例中为金额)，但也呈现JSX(在本例中为货币转换的组件)。

```javascript
import * as React from 'react';
const App = () => (
  <div>
    <h1>US Dollar to Euro:</h1>
    <Amount toCurrency={(amount) => <Euro amount={amount} />} />
    <h1>US Dollar to Pound:</h1>
    <Amount toCurrency={(amount) => <Pound amount={amount} />} />
  </div>
);
const Amount = ({ toCurrency }) => {
  const [amount, setAmount] = React.useState(0);
  const handleIncrement = () => setAmount(amount + 1);
  const handleDecrement = () => setAmount(amount - 1);
  return (
    <div>
      <button type="button" onClick={handleIncrement}>
        +
      </button>
      <button type="button" onClick={handleDecrement}>
        -
      </button>
      <p>US Dollar: {amount}</p>
      {toCurrency(amount)}
    </div>
  );
};
const Euro = ({ amount }) => <p>Euro: {amount * 0.86}</p>;
const Pound = ({ amount }) => <p>Pound: {amount * 0.76}</p>;
export default App;
```

其次，重构它，从拥有任意渲染prop到拥有一个更具体的Children函数

```javascript
import * as React from 'react';
const App = () => (
  <div>
    <h1>US Dollar to Euro:</h1>
    <Amount>{(amount) => <Euro amount={amount} />}</Amount>
    <h1>US Dollar to Pound:</h1>
    <Amount>{(amount) => <Pound amount={amount} />}</Amount>
  </div>
);
const Amount = ({ children }) => {
  const [amount, setAmount] = React.useState(0);
  const handleIncrement = () => setAmount(amount + 1);
  const handleDecrement = () => setAmount(amount - 1);
  return (
    <div>
      <button type="button" onClick={handleIncrement}>
        +
      </button>
      <button type="button" onClick={handleDecrement}>
        -
      </button>
      <p>US Dollar: {amount}</p>
      {children(amount)}
    </div>
  );
};
const Euro = ({ amount }) => <p>Euro: {amount * 0.86}</p>;
const Pound = ({ amount }) => <p>Pound: {amount * 0.76}</p>;
export default App;
```

基本上这就是区分渲染prop和更具体的子元素作为一个函数(其核心也是渲染prop)的所有内容了。前者被当作任何元素的prop，后者被当作Children的prop。你之前已经看到过，这个函数可以作为回调处理程序(例如按钮)传递给 React 组件，但是这一次这个函数被传递给实际渲染的东西，而渲染的职责部分地移到渲染prop组件之外，而prop则由渲染prop组件本身提供。

你可以在 [GitHub](https://github.com/the-road-to-learn-React/React-children-as-a-function-example) 上找到这个小项目。再次强调，如果您在上一个例子之后遇到了任何问题，请查看引用的文章，因为本指南没有详细介绍 React 中渲染prop组件的细节。
