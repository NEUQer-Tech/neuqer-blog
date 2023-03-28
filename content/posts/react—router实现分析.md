---
title: "React—router实现分析"
date: 2023-03-27T20:54:48+08:00
draft: false
author: 柯昌鹏
tags: ["技术分享", "React"]
authorLink: "https://github.com/kcfuler"
---

## 3.22

> 主要内容：
>
> - 浏览器内置的导航相关 api
> - history 库介绍
> - react-router@6.22部分组件的实现介绍

# 前置知识

## 浏览器 api

> 我们在浏览器中实现的任何功能，底层一定都有对应的 api 支持。理解一个功能依赖的底层 api 是理解它的实现的第一步

react-router 所依赖的浏览器 api 主要是以下两个

- window.history
- window.location.hash

下面依次介绍

### window.history

`window.history` 是 JavaScript DOM（文档对象模型）中的一个对象，通过 History API 提供对浏览器会话历史记录的访问。History API 允许开发者操作浏览器的历史记录堆栈，其中包含用户访问过的 URL。

`window.history` 中我们主要用到的是以下两个方法

- `window.history.pushState()`：向会话历史记录**添加**新状态，并更新当前页面的 URL。
- `window.history.replaceState()`：使用新状态更新会话历史记录的当前状态，而**不创建**历史堆栈中的新条目。

#### 对应事件

`window.onpopstate` ， 我们一般通过监听这个事件来实现导航对应的页面功能

需要注意的是，在同一文档的两个记录条目之间会触发该事件，而 `history.pushState` 和 `histroy.replaceState()`都不会触发 `popstate` 事件

我们在使用 history 模式下的路由时就会使用上述的几个 api 来进行路由的实现。

### window.location.hash

这个属性表示的就是在一个路径中，#后面的内容

```HTML
<a id="myAnchor" href="/zh-CN/docs/Location.href#Examples">Examples</a>
<script>
  var anchor = document.getElementById("myAnchor");
  console.log(anchor.hash); // 返回'#Examples'
</script>
```

我们可以通过 `window.location.hash`来获取到导航栏中 hash 的值，也可以通过上述代码得到一个标签的 hash 值。

#### 对应事件

`hashchange` , 我们可以通过这个事件监听到 hash 的变化，从而完成对路由的响应。

## history 库

它的作用可以引用官方的介绍：

> `history库`可以令你更轻松地管理在`JavaScript`环境下运行的会话历史（`session history`）。一个`history`实例抽象了各种环境中的差异，且提供了最简洁的`API`去管理会话中的历史栈（`history stack`）、导航（`navigate`）以及持久化的状态（`persist state`）。

- history 库是 react-router 底层依赖的库，通过它实现对路径变化的响应。
- history 库主要提供了对上述浏览器 api 的封装，更方便对响应路径的变化

> 详细解读可以阅读文末的参考文章

# 实现原理

## 基础部分

### BrowserRouter

使用 React-router，第一步就是用 `BrowserRouter`来包裹入口组件

```TypeScript
ReactDOM.render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>,
  document.getElementById("root")
);
```

下面通过 Browser 的源码来分析其作用

- 创建路由容器，将路由状态通过参数传递给`Router`

```TypeScript
// packages\react-router-dom\index.tsx
export function BrowserRouter({
  basename, // 路由统一的前缀
  children,
  window, // 指明要监控和操作哪个页面对象的路由变化。默认是window对象，但我们可以传入iframe对象
}: BrowserRouterProps) {
  // 创建ref对象historyRef,用于存储createBrowserHistory创建出来的会话历史管理实例
  let historyRef = React.useRef<BrowserHistory>();
  if (historyRef.current == null) {
    historyRef.current = createBrowserHistory({ window });
  }

  let history = historyRef.current;
  let [state, setState] = React.useState({
    /*
     * 到达当前路由的行为，有以下值：
     *    "POP": 代表路由的变化是通过 history.go这类操作历史栈索引的API 或者 浏览器导航栏上的前进和后退键触发。
     *    "PUSH": 代表路由的变化是通过 history.push 触发的
     *    "REPLACE": 代表路由的变化是通过 history.replace 触发的
     */
    action: history.action,
    /**
     * 一个记录当前路由数据的对象，包含以下属性：
     *    pathname：等同于window.location.pathname
     *    search：等同于window.location.search
     *    hash：等同于window.location.hash
     *    state：当前路由地址的状态，类似但不等于window.history.state
     *    key：代表当前路由地址的唯一值
     */
    location: history.location,
  });
  /*
   *   useEffect 与 useLayoutEffect的区别：
   *   useEffect: 在react执行commit后，也就是页面渲染变化后执行
   *   useLayoutEffect: 在react执行commit前执行，会阻塞页面渲染变化
   */
  React.useLayoutEffect(() => history.listen(setState), [history]);
  // 用Router组件传入一些参数且包裹着children返回出去
  return (
    <Router
      basename={basename}
      children={children}
      location={state.location}
      navigationType={state.action}
      navigator={history}
    />
  );
}
```

### Router 组件

```TypeScript
// packages\react-router\index.tsx
export function Router({
  basename: basenameProp = "/",
  children = null,
  location: locationProp, // history.location
  navigationType = NavigationType.Pop, // history.action
  navigator, // history
  static: staticProp = false, // 该属性在BrowserRouter上用不到
}: RouterProps): React.ReactElement | null {
  // normalizePathname用于对basename格式化，如normalizePathname('//asd/')=>'/asd'
  let basename = normalizePathname(basenameProp);
  let navigationContext = React.useMemo(
    () => ({ basename, navigator, static: staticProp }),
    [basename, navigator, staticProp]
  );
  // 如果locationProp为字符串则把他转为对象(包含pathname,search,state,key,hash)
  if (typeof locationProp === "string") {
    locationProp = parsePath(locationProp);
  }

  let {
    pathname = "/",
    search = "",
    hash = "",
    state = null,
    key = "default",
  } = locationProp;
  // 此 location 与 locationProp 的区别在于pathname做了去掉basename的处理
  let location = React.useMemo(() => {
    // trailingPathname为当前location.pathname中截掉basename的那部分，如
    // stripBasename('/prefix/a', '/prefix') => '/a'
    // 如果basename为'/'，则不对pathname处理直接返回原值
    let trailingPathname = stripBasename(pathname, basename);

    if (trailingPathname == null) {
      return null;
    }

    return {
      pathname: trailingPathname,
      search,
      hash,
      state,
      key,
    };
  }, [basename, pathname, search, hash, state, key]);

  if (location == null) {
    return null;
  }
  // 最后返回被NavigationContext和LocationContext包裹着的children出去
  return (
    <NavigationContext.Provider value={navigationContext}>
      <LocationContext.Provider
        children={children}
        value={{ location, navigationType }}
      />
    </NavigationContext.Provider>
  );
}
```

![](/images/react-router-1.png)

### useRoutes

我们先使用 useRoutes 来改写我们的`App`

```TypeScript
// 为了演示不同路由的效果，新建了Route1和Route2组件
const Route1 = () => <div>Route1</div>;
const Route2 = () => <div>Route2</div>;
const Page = () => <div>Page</div>;

const App = () => {
  // 通过把路由规则对象传入useRoutes，返回的是根据路由规则生成的Routes
  // 如果按照我们下面传的路由规则对象，则生成的Routes如下所示
  /**
   *  <Routes>
   *    <Route path="/" element={<Page />}/>
   *    <Route path="/route1" element={<Route1 />} />
   *    <Route path="/route2" element={<Route2 />} />
   *  </Routes>
   */
  const element = useRoutes([
    { path: "/", element: <Page /> },
    { path: "/route1", element: <Route1 /> },
    { path: "/route2", element: <Route2 /> },
  ]);

  return <div>{element}</div>;
};

export default App;
```

可以达到和之前的写法一样的效果。

那么，这样的效果是如何达到的呢，我们可以以这个例子从 `useRoutes`开始分析

```TypeScript
// packages\react-router\index.tsx
export function useRoutes(
  routes: RouteObject[],
  locationArg?: Partial<Location> | string
): React.ReactElement | null {
  /*
    RouteContext的声明代码为：
      const RouteContext = React.createContext<RouteContextObject>({
        outlet: null,
        matches: [],
      });
    由于在`BrowserRouter`中没有引用 RouteContext 的逻辑，因此此情况下解构取出的matches为空数组
  */
  let { matches: parentMatches } = React.useContext(RouteContext);
  // 此情况下routeMatch为undefined
  let routeMatch = parentMatches[parentMatches.length - 1];
  // 此情况下parentParams为空对象
  let parentParams = routeMatch ? routeMatch.params : {};
  // 此情况下parentPathname为"/"
  let parentPathname = routeMatch ? routeMatch.pathname : "/";
  // 此情况下parentPathnameBase为"/"
  let parentPathnameBase = routeMatch ? routeMatch.pathnameBase : "/";
  // 此情况下parentRoute为false
  let parentRoute = routeMatch && routeMatch.route;
  /*
    useLocation的代码如下所示：
      function useLocation(): Location {
        return React.useContext(LocationContext).location;
      }
    可看出该函数作用是用于取出LocationContext中的location
   */
  let locationFromContext = useLocation();

  // 如果存在传入的locationArg，则此处的location为locationArg，否则是上面的locationFromContext
  let location;
  if (locationArg) {
    let parsedLocationArg =
      typeof locationArg === "string" ? parsePath(locationArg) : locationArg;
    location = parsedLocationArg;
  } else {
    location = locationFromContext;
  }

  let pathname = location.pathname || "/";
  // 从location.pathname中截取父路由的pathname部分
  let remainingPathname =
    parentPathnameBase === "/"
      ? pathname
      : pathname.slice(parentPathnameBase.length) || "/";
  /*
    matchRoutes的TyprScript声明如下所示：
      declare function matchRoutes(
        routes: RouteObject[],
        location: Partial<Location> | string,
        basename?: string
      ): RouteMatch[] | null;
    该函数会从routes中找出所有匹配location的路由（包括父子路由），然后组成RouteMatch[]格式的数组返回出去
    RouteMatch的TyprScript声明如下所示：
      interface RouteMatch<ParamKey extends string = string> {
        params: Params<ParamKey>;
        pathname: string;
        route: RouteObject;
      }
    此函数的源码由于比较复杂，所以被放在源码分析（深入部分）的内容里分析：
   */
  let matches = matchRoutes(routes, { pathname: remainingPathname });
  /*
    _renderMatches其实就是renderMatches，其声明类型如下所示：
      declare function renderMatches(
        matches: RouteMatch[] | null
      ): React.ReactElement | null;
    其用于把 matchRoutes 函数返回的结果渲染成 React.ReactElement
    */
  return _renderMatches(
    // 对matches进行增强处理
    matches &&
      matches.map((match) =>
        Object.assign({}, match, {
          params: Object.assign({}, parentParams, match.params),
          pathname: joinPaths([parentPathnameBase, match.pathname]),
          pathnameBase:
            match.pathnameBase === "/"
              ? parentPathnameBase
              : joinPaths([parentPathnameBase, match.pathnameBase]),
        })
      ),
    parentMatches
  );
}
```

接下来就详细分析`useRoutes`中的`_renderMatches`函数。在路径名为`"/route1"`的情况下，此时`remainingPathname`为`"/route1"`，从 `matchRoutes(routes, { pathname: remainingPathname })`中得出`matches`的值如下所示:

```TypeScript
[
  {
    params: {},
    pathname: "/route1",
    pathnameBase: "/route1",
    route: {
      element: <Route1 />,
      path: "/route1",
    },
  },
];
```

将`matches`带入到 `_renderMatches` 中进行分析：

```TypeScript
function _renderMatches(
  matches: RouteMatch[] | null,
  parentMatches: RouteMatch[] = []
): React.ReactElement | null {
  if (matches == null) return null;

  return matches.reduceRight((outlet, match, index) => {
    return (
      <RouteContext.Provider
        // 此情况下该值为<Route1/>
        children={
          match.route.element !== undefined ? match.route.element : outlet
        }
        /*
          此情况下该值为:
            {
              outlet: null,
              matches: [
                {
                  params:{},
                  pathname: "/route1",
                  pathnameBase: "/route1",
                  route: {
                    element: <Route1/>,
                    path:"/route1"
                  }
                }
              ]
            }
        */
        value={{
          outlet,
          matches: parentMatches.concat(matches.slice(0, index + 1)),
        }}
      />
    );
  }, null as React.ReactElement | null);
}
```

`useRoutes`主要做了以下两件事：

1.  根据当前的路由`location`,从传入的`routes`中找出所有匹配的路由对象，放到数组`matches`里
2.  用`renderMatches`把`matches`渲染成一个`React`元素，期间会把`macthes`从尾到头遍历用`RouteContext`包裹起来，如下所示：

![](/images/react-router-2.png)

如果路由规则对象没有定义 `element` 属性，则 `RouteContext.Provider`的 children 会指向其自身的 `outlet`:

![](/images/react-router-3.png)

### Outlet

上述情况都是在每个子路由里都没有子路由的情况下分析的。我们修改`App`的代码继续分析

```TypeScript
// Route1增加Outlet组件用于渲染匹配子路由对应的组件
const Route1 = () => (
  <div>
    Route1
    <Outlet />
  </div>
);
const Route2 = () => <div>Route2</div>;
const Page = () => <div>Page</div>;

const App = () => {
  const element = useRoutes([
    { path: "/", element: <Page /> },
    {
      path: "/route1",
      element: <Route1 />,
      // 新增子路由规则
      children: [
        { path: "name1", element: <div>name1</div> },
        { path: "name2", element: <div>name2</div> },
      ],
    },
    { path: "/route2", element: <Route2 /> },
  ]);

  return <div>{element}</div>;
};
```

由前述可知，当路径名为`'/route1/name1'`时，在`useRoutes`中，`matchRoutes(routes, { pathname: remainingPathname })`中得出`matches`的值如下所示:

```TypeScript
[
  {
    params: {},
    pathname: "/route1",
    pathnameBase: "/route1",
    route: {
      path: "/route1",
      element: <Route1 />,
      children: [
        { path: "name1", element: <div>name1</div> },
        { path: "name2", element: <div>name2</div> },
      ],
    },
  },
  {
    params: {},
    pathname: "/route1/name1",
    pathnameBase: "/route1/name1",
    route: {
      path: "name1",
      element: <div>name1</div>,
    },
  },
];
```

此时`useRoutes`返回的`React`元素如下所示：

![](/images/react-router-4.png)

基于上述内容，分析`Outlet`源码：

```TypeScript
// packages\react-router\index.tsx
export function Outlet(props: OutletProps): React.ReactElement | null {
  return useOutlet(props.context);
}
```

`Outlet`直接调用了`useOutlet`的结果，继续分析 `useOutlet`

```TypeScript
// packages\react-router\index.tsx
export function useOutlet(context?: unknown): React.ReactElement | null {
  let outlet = React.useContext(RouteContext).outlet;
  if (outlet) {
    return (
      <OutletContext.Provider value={context}>{outlet}</OutletContext.Provider>
    );
  }
  return outlet;
}
```

这里可以看出，其实 `<Outlet/>`就是把当前 `RouterContext`的 `outlet`值渲染出来，如下所示

![](/images/react-router-5.png)

### useNavigate

在 `App` 和 `Route1` 中加入能**更改路径名**的按钮，如下所示：

```TypeScript
const Route1 = () => {
  // 获取能改变路由的navigate函数
  const navigate = useNavigate();
  // 用于改变路由的函数
  const pushPage = (pathname) => {
    navigate(pathname);
  };

  return (
    <div>
      Route1
      <div>
        <button onClick={() => pushPage({ pathname: "name1" })}>/name1</button>
        <button onClick={() => pushPage({ pathname: "name2" })}>/name2</button>
      </div>
      <Outlet />
    </div>
  );
};
const Route2 = () => <div>Route2</div>;
const Page = () => <div>Page</div>;

const App = () => {
  const element = useRoutes([
    { path: "/", element: <Page /> },
    {
      path: "/route1",
      element: <Route1 />,
      // 新增子路由规则
      children: [
        { path: "name1", element: <div>name1</div> },
        { path: "name2", element: <div>name2</div> },
      ],
    },
    { path: "/route2", element: <Route2 /> },
  ]);
  // 获取能改变路由的navigate函数
  const navigate = useNavigate();
  // 用于改变路由的函数
  const pushPage = (pathname: string) => {
    navigate(pathname);
  };

  return (
    <div>
      <button onClick={() => pushPage("/")}>/</button>
      <button onClick={() => pushPage("/route1")}>/route1</button>
      <button onClick={() => pushPage("/route1/name1")}>/route1/name1</button>
      <button onClick={() => pushPage("/route1/name2")}>/route1/name2</button>
      <button onClick={() => pushPage("/route2")}>/route2</button>

      {element}
    </div>
  );
};
```

至于为什么要在`Route1`和`App`里都用上`useNavigate`，是因为`useNavigate`在**路由函数组件**(被`RouteContext.Provider`包裹的组件，如`<Route1/>`)和**非路由函数组件**（如`<App/>`）有不同的执行逻辑。这样子有利于我们在分析`useNavigate`源码时做横向对比。

`navigate`有如下特点

1.  `navigate`可以传入对象或者字符串，都可以达到改变路由的效果
2.  `navigate`传入对象时，其`pathname`实相对于当前路由的`pathname`变化的

下面来分析 `useNavigate` 内部的执行逻辑

```TypeScript
export function useNavigate(): NavigateFunction {
  // 在BrowserRouter中，用NavigationContext包裹着children，
  // 其中basename为BrowserRouter中的basename，navigator为createBrowserHistory创建出的history
  let { basename, navigator } = React.useContext(NavigationContext);
  // 在<App/>中，matches是空数组。
  // 在<Route1/>中，matches是[ RouteMatch(对应Route1) ]
  let { matches } = React.useContext(RouteContext);
  // 把hsitory.location.pathname的值赋给locationPathname
  let { pathname: locationPathname } = useLocation();
  // 在<App/>中，routePathnamesJson为"[]"。
  // 在<Route1/>中，routePathnamesJson为'["/route1"]'。
  // 这里用字符串而非数组的格式是为了下面由useCallback生成的navigate不会因为其值的变化而变化
  let routePathnamesJson = JSON.stringify(
    matches.map((match) => match.pathnameBase)
  );
  // 标志位，在初次渲染之前activeRef为false，其余情况为true
  let activeRef = React.useRef(false);
  React.useEffect(() => {
    activeRef.current = true;
  });

  let navigate: NavigateFunction = React.useCallback(
    (to: To | number, options: NavigateOptions = {}) => {
      // 如果调度所在的函数组件还没初次渲染，则不走下面的流程
      if (!activeRef.current) return;
      // 如果传入参数是数字类型数据如navigate(-1)，则调用go函数(即history.go)
      if (typeof to === "number") {
        navigator.go(to);
        return;
      }
      /*
        resolveTo的作用可以参考useResolvedPath(https://reactrouter.com/docs/en/v6/api#useresolvedpath),
        useResolvedPath内部其实也是调用了resolveTo。其作用在于把to变量转换为数据结构为
        {hash:string,search:string,pathname:string}的path变量，以作为形参被navigate.push和navigate.replace调用。
        关于resolveTo的详细分析我在分析完 useNavigate 后的内容里补充呈现。
      */
      let path = resolveTo(
        to,
        JSON.parse(routePathnamesJson),
        locationPathname
      );

      if (basename !== "/") {
        // const joinPaths = (paths: string[]): string => paths.join("/").replace(/\/\/+/g, "/");
        path.pathname = joinPaths([basename, path.pathname]);
      }
      // 根据navigate中的第二形参是否带{repalce: true}来决定调用history.push还是history.replace
      (!!options.replace ? navigator.replace : navigator.push)(
        path,
        options.state
      );
    },
    [basename, navigator, routePathnamesJson, locationPathname]
  );

  return navigate;
}
```

从`useNavigate`得知，其内部逻辑主要用于处理`to`（即第一形参）的格式：

1.  如果`to`是`number`类型数据，则调用`history.go`处理
2.  如果`to`是`string`或`object`类型数据，则通过`resolveTo`把其转换为`Path`类型的数据然后调用`history.push`或`history.replace`处理。

而`navigate`自身只是用于无刷新地改变路由。但因为在`BrowserRouter`中有这部分逻辑：

```TypeScript
export function BrowserRouter({
  basename,
  children,
  window,
}: BrowserRouterProps) {
  // ...省略其他代码
  React.useLayoutEffect(() => history.listen(setState), [history]);
  // ...省略其他代码
}
```

而`history.go`、`hsitory.push`、`hsitory.replace`在执行后都会触发执行`history.listen`中注册的函数，而`setState`的执行会让`BrowserRouter`及其`children`更新，从而让页面响应式变化。

现在就来看看 useNavigate 中用到的 resolveTo 函数， 从 TypeScript 的语法来看，该函数是把传入的变量从 To 声明类型转换成 Path 声明类型。To 和 Path 的声明类型如下所示：

```TypeScript
export declare type To = string | PartialPath;
type PartialPath = Partial<Path>

export interface Path {
  pathname: Pathname;
  search: Search;
  hash: Hash;
}
export declare type Pathname = string;
export declare type Search = string;
export declare type Hash = string;
```

**除了转换成\*\***`Path`\***\*声明类型的数据**，`resolveTo`函数还有一个更重要的作用：**重点处理\*\***`to.pathname`\***\*中涉及到相对路径的写法**。

```TypeScript
// packages\react-router\index.tsx
function resolveTo(
  toArg: To,
  // 即matches.map((match) => match.pathnameBase)，macthes从当前RouteContext中取出
  routePathnames: string[],
  // 即location.pathname
  locationPathname: string
): Path {
  // parsePath引用自history库，用于把路径字符串转换为Path类型的数据
  let to = typeof toArg === "string" ? parsePath(toArg) : toArg;
  let toPathname = toArg === "" || to.pathname === "" ? "/" : to.pathname;

  // 官方注释：
  //   If a pathname is explicitly provided in `to`, it should be relative to the
  //   route context. This is explained in `Note on `<Link to>` values` in our
  //   migration guide from v5 as a means of disambiguation between `to` values
  //   that begin with `/` and those that do not. However, this is problematic for
  //   `to` values that do not provide a pathname. `to` can simply be a search or
  //   hash string, in which case we should assume that the navigation is relative
  //   to the current location's pathname and *not* the route pathname.
  // 从官方注释可知：
  //    1. 如果to.pathname被定义了，则该值是相对于当前路由上下文去运行的。
  //    2. 而且存在一种情况是to没有定义pathname而是定义了hash或search，这种情况下也是可以运行的，此时会基于当前路由而变化
  // 由于to.pathname可以用相对路径的写法。因此需要from记录把哪个路由作为起点进行跳转的，
  // 这里把from记录的路由成为“基准路由”
  let from: string;
  if (toPathname == null) {
    from = locationPathname;
  } else {
    // routePathnameIndex用于记录“基准路由”是取自routePathnames的第几个元素
    let routePathnameIndex = routePathnames.length - 1;

    if (toPathname.startsWith("..")) {
      let toSegments = toPathname.split("/");

      // Each leading .. segment means "go up one route" instead of "go up one
      // URL segment".  This is a key difference from how <a href> works and a
      // major reason we call this a "to" value instead of a "href".
      // 从官方注释可知，".."代表以父路由路径名为基准进行跳转。可以有多个".."合并使用例如"../../"
      // 处理to.pathname中的".."情况，每当存在一个".."，则routePathnameIndex减1
      while (toSegments[0] === "..") {
        toSegments.shift();
        routePathnameIndex -= 1;
      }

      to.pathname = toSegments.join("/");
    }

    // If there are more ".." segments than parent routes, resolve relative to
    // the root / URL.
    // 如果to.pathname中的".."太多导致routePathnameIndex<0，则from取根目录
    from = routePathnameIndex >= 0 ? routePathnames[routePathnameIndex] : "/";
  }
  /*
    resolvePath是一个对外export的API（https://reactrouter.com/docs/en/v6/api#resolvepath）
    其声明类型如下：
      declare function resolvePath(
        to: To,
        fromPathname?: string
      ): Path;
    其作用在于根据to和from生成一个pathname为绝对路径的Path类型变量，
    为什么需要pathname为绝对路径？
      因为打最后我们调用navigator.push或navigator.replace时，传入的Path类型变量的pathname只能是绝对路径
      react-router中的navigate支持其形参中pathname为相对路径或绝对路径，
      但history.push和history.replace只支持其形参中的pathname为绝对路径
  */
  let path = resolvePath(to, from);

  // Ensure the pathname has a trailing slash if the original to value had one.
  if (
    toPathname &&
    toPathname !== "/" &&
    toPathname.endsWith("/") &&
    !path.pathname.endsWith("/")
  ) {
    path.pathname += "/";
  }

  return path;
}
```

### Link

我们继续在上面的 App 的基础上去修改，把 `<button/>` 改为 `<Link />`， 如下所示：

```TypeScript
const Route1 = () => {
  return (
    <div>
      Route1
      <div>
        {/* <button onClick={() => pushPage({ pathname: 'name1' })}>/name1</button>
        <button onClick={() => pushPage({ pathname: 'name2' })}>/name2</button> */}
        {/*换成Link*/}
        <Link to={{ pathname: "name1" }}>{`{pathname: 'name1'}`}</Link>&emsp;
        <Link to={{ pathname: "name2" }}>{`{pathname: 'name2'}`}</Link>
      </div>
      <Outlet />
    </div>
  );
};
const Route2 = () => <div>Route2</div>;
const Page = () => <div>Page</div>;

const App = () => {
  const element = useRoutes([
    { path: "/", element: <Page /> },
    {
      path: "/route1",
      element: <Route1 />,
      // 新增子路由规则
      children: [
        { path: "name1", element: <div>name1</div> },
        { path: "name2", element: <div>name2</div> },
      ],
    },
    { path: "/route2", element: <Route2 /> },
  ]);

  return (
    <div>
      {/* <button onClick={() => pushPage('/')}>/</button>
      <button onClick={() => pushPage('/route1')}>/route1</button>
      <button onClick={() => pushPage('/route1/name1')}>/route1/name1</button>
      <button onClick={() => pushPage('/route1/name2')}>/route1/name2</button>
      <button onClick={() => pushPage('/route2')}>/route2</button> */}
      {/*换成Link*/}
      <Link to="/">/</Link>&emsp;
      <Link to="/route1">/route1</Link>&emsp;
      <Link to="/route1/name1">/route1/name1</Link>&emsp;
      <Link to="/route1/name2">/route1/name2</Link>&emsp;
      <Link to="/route2">/route2</Link>&emsp;
      {element}
    </div>
  );
};
```

接下来我们基于上面的例子来学习 `Link` 的源码，在学习之前，我们需要先了解 `Link` 源码中重点用到的两个函数 `useHref` 和 `useLinkClickHandler`

- useHref

```TypeScript
// packages\react-router\index.tsx
/*
  https://reactrouter.com/docs/en/v6/api#usehref
  官方解释：该API用于根据给定的to变量生成一个可以跳转到指定路由的URL字符串
*/
export function useHref(to: To): string {
  let { basename, navigator } = React.useContext(NavigationContext);
  /*
    https://reactrouter.com/docs/en/v6/api#useresolvedpath
    useResolvedPath的作用和resolveTo一样（useResolvedPath内部就是调用了resolveTo且返回该函数的处理结果）
    即根据给定的to变量返回Path类型的变量，其中会处理to.pathname的相对路径写法。
  */
  let { hash, pathname, search } = useResolvedPath(to);

  let joinedPathname = pathname;
  if (basename !== "/") {
    /*
      getToPathname作用在于从给定to获取pathname，源码如下所示：
        function getToPathname(to: To): string | undefined {
          // Empty strings should be treated the same as / paths
          return to === "" || (to as Path).pathname === ""
            ? "/"
            : typeof to === "string"
            ? parsePath(to).pathname
            : to.pathname;
        }
    */
    let toPathname = getToPathname(to);
    let endsWithSlash = toPathname != null && toPathname.endsWith("/");
    joinedPathname =
      pathname === "/"
        ? basename + (endsWithSlash ? "/" : "")
        // const joinPaths = (paths: string[]): string => paths.join("/").replace(/\/\/+/g, "/");
        : joinPaths([basename, pathname]);
  }
  // 使用history.createHref(https://github.com/remix-run/history/blob/main/docs/api-reference.md#historycreatehrefto-to)把To类型变量转变成URL字符串，
  // 因为hsitory.createHref不具备支持basename和to.pathname的相对路径写法。因此有了上面的处理这两者的逻辑
  return navigator.createHref({ pathname: joinedPathname, search, hash });
}
```

- useLinkClickHandler

```TypeScript
// packages\react-router-dom\index.tsx
/*
  https://reactrouter.com/docs/en/v6/api#uselinkclickhandler
  官方解释：该API用于生成一个用于导航的点击事件，这个点击事件用于自定义的`<Link/>`组件
*/
export function useLinkClickHandler<E extends Element = HTMLAnchorElement>(
  to: To,
  {
    target,
    replace: replaceProp,
    state,
  }: {
    // React.HTMLAttributeAnchorTarget声明类型：type HTMLAttributeAnchorTarget = '_self' | '_blank' | '_parent' | '_top'  | (string & {});（与a标签的target一样）
    target?: React.HTMLAttributeAnchorTarget;
    // 定义跳转的行为是PUSH还是REPLACE
    replace?: boolean;
    // 跳转的时候可以在此定义即将跳转的路由的location.state
    state?: any;
  } = {}
): (event: React.MouseEvent<E, MouseEvent>) => void {
  let navigate = useNavigate();
  let location = useLocation();
  // useResolvedPath在上面已经介绍过了，这里就不再重复介绍了
  let path = useResolvedPath(to);

  return React.useCallback(
    (event: React.MouseEvent<E, MouseEvent>) => {
      if (
        /*
          MouseEvent.button是只读属性，它返回一个number类型值来代表什么键被操作，例如：
            0：主按键，通常指鼠标左键或默认值（译者注：如document.getElementById('a').click()这样触发就会是默认值）
            1：辅助按键，通常指鼠标滚轮中键
          更多内容可看：https://developer.mozilla.org/zh-CN/docs/Web/API/MouseEvent/button
        */
        event.button === 0 &&
        // 如果target已被定义且并非_self值，则执行默认事件
        (!target || target === "_self") &&
        /*
          isModifiedEvent用于检测该事件是否是鼠标 + 键盘键一并触发
          isModifiedEvent源码：
            function isModifiedEvent(event: React.MouseEvent) {
              return !!(event.metaKey || event.altKey || event.ctrlKey || event.shiftKey);
            }
        */
        !isModifiedEvent(event)
      ) {
        // 阻止a标签被点击后默认事件的执行
        event.preventDefault();

        /*
          replace变量用于决定此次跳转行为是PUSH还是REPLACE，
          其中createPath(location) === createPath(path)的逻辑是，如果新路由的url与当前路由的一致，则使用REPLACE
          此处createPath其实是history.createPath，用于给定的Partial<Path>类型的变量生成URL字符串，官方地址：
            https://github.com/remix-run/history/tree/main/docs/api-reference.md#createpath
        */
        let replace = !!replaceProp || createPath(location) === createPath(path);

        navigate(to, { replace, state });
      }
    },
    [location, navigate, path, replaceProp, state, target, to]
  );
}
```

在有了上面两个函数之后，我们就可以来分析 `Link` 组件的代码了

```TypeScript
// packages/react-router-dom/index.tsx
export const Link = React.forwardRef<HTMLAnchorElement, LinkProps>(
  function LinkWithRef(
    { onClick, reloadDocument, replace = false, state, target, to, ...rest },
    ref
  ) {
    let href = useHref(to);
    let internalOnClick = useLinkClickHandler(to, { replace, state, target });
    function handleClick(
      event: React.MouseEvent<HTMLAnchorElement, MouseEvent>
    ) {
      // 如果Link中还定义了onClick，则先执行onClick中定义的事件
      if (onClick) onClick(event);
      // event.defaultPrevented 返回一个布尔值，表明当前事件是否调用了 event.preventDefault()方法。
      // 因为onClick定义的事件里可能调用了event.preventDefault，因此这里做个判断
      if (!event.defaultPrevented && !reloadDocument) {
        internalOnClick(event);
      }
    }

    return (
      <a
        {...rest}
        href={href}
        onClick={handleClick}
        ref={ref}
        target={target}
      />
    );
  }
);
```

当<Link/> 被点击时，它会调用 `navigate` 去跳转路由（除非指定特定参数使其不跳转），而从`useNavigate`的分析中，我们知道调用`navigate`会间接的触发页面响应式更新，也就实现了路由的跳转功能。

## 总结

简单来说，react-router 的实现主要需要以下几点：

- `history`库，用于监听和表示浏览器的路径状态
- 通过`Context`来实现父子组件之间数据的传递，这里传递的是路由的信息和路由方法
- 通过顶层的`setState`来实现响应路径变化，并触发页面的更新
- 通过高阶组件来完成各个组件之间的逻辑传递和拓展
- 对各种边界情况的判断和处理

# 参考文章

> [搞不懂路由跳转?带你了解 history.js 实现原理 - 掘金](https://juejin.cn/post/7192479334962528317#comment)
>
> [History API - Web API 接口参考 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/History_API)
>
> [搞不懂路由跳转?带你了解 history.js 实现原理 - 掘金](https://juejin.cn/post/7192479334962528317#comment)
>
> [react-routerV6 依赖的 history 库源码分析 - 掘金](https://juejin.cn/post/7021799679616614436)
>
> [简单易懂地剖析 React-RouterV6 源码(代码示例+画图总结) - 掘金](https://juejin.cn/post/7075146381907722276)
