# styled-components 在 qiankun 中样式未更新问题排查

## TLDR

由于 styled-components 使用 CSSOM 的方式进行样式插入，在子应用切换后，虽然对应的 HTML Style Element 被*qiankun*恢复到当前 DOM 当中，但是由于恢复后 styled-components 中对应的 sheet 被清除掉了，导致后续样式均无法再添加而导致这个问题。目前比较理想的方法可以通过在应用外包裹上 StyleSheetManager 并传入对应的 target 为 子应用的 container，即可解决此问题，示例代码如下：

```tsx
import { StyleSheetManager } from "styled-components";

function mount({ container }: { container: HTMLElement }) {
  React.render(
    <StyleSheetManager target={container}>
      <App />
    </StyleSheetManager>,
    container.querySelector("#root")
  );
}
```

## 问题复现

具体问题复现可以参照[qiankun-styled-components-app-switch-solution](https://github.com/wanglam/qiankun-styled-components-app-switch-solution) 这个项目。在该项目中，我们定义了 3 个子应用，每个应用每秒钟刷新一次显示出当前时间。在切换 app1 / app2 切换后会出现样式丢失，所切换的时间不再显示等问题。在该文章中我们会具体分析问题出现的原因及相关解决方案。

## 问题分析

### 现状分析

```tsx
const Clock = styled.div<{ dateTime: string }>`
  &::after {
    ${({ dateTime }) => `content: "${dateTime}";`}
  }
`;

function App() {
  const [dateTime, setDatetime] = useState("");

  setInterval(() => {
    setDatetime(new Date().toString());
  }, 1000);
  return (
    <>
      <h1>Sub App 1</h1>
      <Navigation />
      <Clock
        style={{ background: "skyblue", padding: "4px 10px", color: "white" }}
        dateTime={dateTime}
      />
      <div>Date Time: {dateTime}</div>
    </>
  );
}
```

上述代码是 bug 重现库中实际运行的代码，在样式正常更新的时候我们可以看到，*Clock*元素的 class 属性 每秒钟都会由 styled-components 生成一个新的，并且插入在当前 DOM 下的 style 标签当中。在切换子应用后会发现，*Clock*元素的 class 属性依然在更新，但是所更新的 class 缺没有生成插入对应的样式。由此可见样式未更新的原因很可能是 styled-components 在插入样式时出现了问题。那么我们先大致看以下 styled-components 是使样式生效的呢？

### styled-components 插入样式表的逻辑

```ts
// Copy from https://github.com/styled-components/styled-components/blob/404190e26062133a7b916d73c1d52312db20bc92/packages/styled-components/src/models/StyledComponent.js#L83
function useInjectedStyle<T>(
  componentStyle: ComponentStyle,
  isStatic: boolean,
  resolvedAttrs: T,
  warnTooManyClasses?: $Call<typeof createWarnTooManyClasses, string, string>
) {
  const styleSheet = useStyleSheet();
  const stylis = useStylis();

  const className = isStatic
    ? componentStyle.generateAndInjectStyles(EMPTY_OBJECT, styleSheet, stylis)
    : componentStyle.generateAndInjectStyles(resolvedAttrs, styleSheet, stylis);

  // eslint-disable-next-line react-hooks/rules-of-hooks
  if (process.env.NODE_ENV !== "production") useDebugValue(className);

  if (
    process.env.NODE_ENV !== "production" &&
    !isStatic &&
    warnTooManyClasses
  ) {
    warnTooManyClasses(className);
  }

  return className;
}
```

上述代码中，styled-components 生成的样式组件使用 useInjectedStyle 将组件样式插入到样式表当中。通过调用 useStyleSheet 来获取当前使用的样式表。

```ts
// Copy from https://github.com/styled-components/styled-components/blob/404190e26062133a7b916d73c1d52312db20bc92/packages/styled-components/src/models/StyleSheetManager.js#L25
export function useStyleSheet(): StyleSheet {
  return useContext(StyleSheetContext) || masterSheet;
}
```

上述代码中首先使用 StyleSheetContext 中已经存在的样式表，如果对应的样式表不存在的话则会使用 masterSheet（这点很重要，出现问题的原因和这一点很有关系）。一般来说不在 App 外面包裹 StyleSheetManager 的时候都是使用 masterSheet。

```ts
// Copy from https://github.com/styled-components/styled-components/blob/404190e26062133a7b916d73c1d52312db20bc92/packages/styled-components/src/models/StyleSheetManager.js#L22
export const masterSheet: StyleSheet = new StyleSheet();
```

从上述代码中不难看出，masterSheet 是 StyleSheet 这个类的一个实例。

```ts
// Copy from https://github.com/styled-components/styled-components/blob/404190e26062133a7b916d73c1d52312db20bc92/packages/styled-components/src/sheet/Sheet.js#L75
  getTag(): GroupedTag {
    return this.tag || (this.tag = makeGroupedTag(makeTag(this.options)));
  }
```

而在该实例中使用 makeTag 来获取真实储存样式内容的节点。

```ts
// Copy from https://github.com/styled-components/styled-components/blob/404190e26062133a7b916d73c1d52312db20bc92/packages/styled-components/src/sheet/Tag.js#L8
export const makeTag = ({
  isServer,
  useCSSOMInjection,
  target,
}: SheetOptions): Tag => {
  if (isServer) {
    return new VirtualTag(target);
  } else if (useCSSOMInjection) {
    return new CSSOMTag(target);
  } else {
    return new TextTag(target);
  }
};
```

而从 makeTag 的代码中可以看出，根据 StyleSheet 的不同创建选项，将使用不同的的 Tag 来储存具体的样式，当 isServer 为 true 时，使用 VirtualTag(与目前的问题暂无关联)。当 useCSSOMInjection 时，使用 CSSOMTag。其他使用使用 TextTag。

```ts
// Copy from https://github.com/styled-components/styled-components/blob/404190e26062133a7b916d73c1d52312db20bc92/packages/styled-components/src/sheet/Tag.js#L37
 insertRule(index: number, rule: string): boolean {
    try {
      this.sheet.insertRule(rule, index);
      this.length++;
      return true;
    } catch (_error) {
      return false;
    }
  }
```

上述代码中展示了 CSSOMTag 插入 CSS Rule 的方法。这里主要使用 HTMLStyleElement 中的 sheet 属性中的 insertRule 方法完成插入操作。

```ts
// Copy from https://github.com/styled-components/styled-components/blob/404190e26062133a7b916d73c1d52312db20bc92/packages/styled-components/src/sheet/Tag.js#L79
 insertRule(index: number, rule: string): boolean {
    if (index <= this.length && index >= 0) {
      const node = document.createTextNode(rule);
      const refNode = this.nodes[index];
      this.element.insertBefore(node, refNode || null);
      this.length++;
      return true;
    } else {
      return false;
    }
  }
```

上述代码中展示了 TextTag 插入 CSS Rule 的方法。这里主要使用了 createTextNode 创建新的文本节点，创建好节点后使用 insertBefore 插入到 HTMLStyleElement 当中。

```ts
import { DISABLE_SPEEDY, IS_BROWSER } from "../constants";

// Copy from https://github.com/styled-components/styled-components/blob/a808f40156e62db5c92b06d362e200e5c172a897/packages/styled-components/src/sheet/Sheet.js#L23
const defaultOptions: SheetOptions = {
  isServer: !IS_BROWSER,
  useCSSOMInjection: !DISABLE_SPEEDY,
};
```

从上述代码中可以看出，makeTag 方法中的默认 useCSSOMInjection 属性由 constants 中的 DISABLE_SPEEDY 常量进行控制。

```ts
// Code from https://github.com/styled-components/styled-components/blob/72a6f642c3dedbcee91a2b5d76f96b93f345b86f/packages/styled-components/src/constants.js#L17
export const DISABLE_SPEEDY = Boolean(
  typeof SC_DISABLE_SPEEDY === "boolean"
    ? SC_DISABLE_SPEEDY
    : typeof process !== "undefined" &&
      typeof process.env.REACT_APP_SC_DISABLE_SPEEDY !== "undefined" &&
      process.env.REACT_APP_SC_DISABLE_SPEEDY !== ""
    ? process.env.REACT_APP_SC_DISABLE_SPEEDY === "false"
      ? false
      : process.env.REACT_APP_SC_DISABLE_SPEEDY
    : typeof process !== "undefined" &&
      typeof process.env.SC_DISABLE_SPEEDY !== "undefined" &&
      process.env.SC_DISABLE_SPEEDY !== ""
    ? process.env.SC_DISABLE_SPEEDY === "false"
      ? false
      : process.env.SC_DISABLE_SPEEDY
    : process.env.NODE_ENV !== "production"
);
```

从上述代码中可以看出，在 不存在全局变量 SC_DISABLE_SPEEDY，以及 环境变量 REACT_APP_SC_DISABLE_SPEEDY 和 SC_DISABLE_SPEEDY 未定义的情况下，只要 NODE_ENV 不为 production，则会开启 DISABLE_SPEEDY 为 false 否则将会设置为 true。即 NODE_ENV 为 production 时，会使用 CSSOMTag 来作为样式的插入对象。

### 在 qiankun 的环境下使用 CSSOMTag 为啥会有问题

根据现象观察，在不切换子应用的场景下，styled-components 插入的样式是能够正常工作的。只有在子应用被销毁再重新加载之后才会有类似的问题。

通过上面的 CSSOMTag 中 insertRule 部分的代码我们可以看到，样式的插入是通过 CSSOMTag 实例中 sheet 属性进行样式插入的操作。

```ts
// Copy from https://github.com/styled-components/styled-components/blob/a808f40156e62db5c92b06d362e200e5c172a897/packages/styled-components/src/sheet/Tag.js#L25

  constructor(target?: HTMLElement) {
    const element = (this.element = makeStyleTag(target));

    // Avoid Edge bug where empty style elements don't create sheets
    element.appendChild(document.createTextNode(''));

    this.sheet = getSheet(element);
    this.length = 0;
  }

```

通过 CSSOMTag 中的构造代码可以看到，sheet 属性是通过 getSheet 方法拿到。

```ts
// Copy from https://github.com/styled-components/styled-components/blob/a808f40156e62db5c92b06d362e200e5c172a897/packages/styled-components/src/sheet/dom.js#L44
export const getSheet = (tag: HTMLStyleElement): CSSStyleSheet => {
  if (tag.sheet) {
    return ((tag.sheet: any): CSSStyleSheet);
  }

  // Avoid Firefox quirk where the style element might not have a sheet property
  const { styleSheets } = document;
  for (let i = 0, l = styleSheets.length; i < l; i++) {
    const sheet = styleSheets[i];
    if (sheet.ownerNode === tag) {
      return ((sheet: any): CSSStyleSheet);
    }
  }

  throwStyledError(17);
  return (undefined: any);
};
```

从 getSheet 中的代码可以看到，如果传过来的 tag 存在 sheet 属性的话则会直接返回 tag 中的 sheet 属性作为返回值。这里我们可以看作 CSSOMTag 中的 sheet 属性实际上用的就是插入的 HTMLStyleElement 中的 sheet。

```ts
(() => {
  const styleTag = document.createElement("style");

  document.head.appendChild(styleTag);

  const sheet = styleTag.sheet;

  document.head.removeChild(styleTag);
  // false
  console.log(sheet === styleTag.sheet);

  document.head.appendChild(styleTag);
  // false
  console.log(sheet === styleTag.sheet);
})();
```

通过上述测试代码我们可以看到，在 DOM 上先删除再重新添加 HTMLStyleElement 后，该元素中的 sheet 属性已经和我们最开始保存的 sheet 的值不一样了，即在该 sheet 中做任何操作也不会再重新反应到当前页面上了。这就解释了 CSSOMTag 中为什么不再插入新的样式了。根据 useStyleSheet 中的代码，在应用切换后 masterSheet 并不会重新初始化。即 CSSOMTag 中的 sheet 一直使用比较旧的值，导致样式再也无法添加。

## 解决方案

根据前面的分析我们大致知道了问题发生的原因，顺着原因我们可以设想在子应用切换后，如果能再重新生成一下 useStyleSheet 中返回的 sheet 应该就能解决问题。在 useStyleSheet 中首先使用 StyleSheetContext 中的值作为返回结果，只有当 context 中的值不存在时才会使用 masterSheet。

```tsx
// Copy from https://github.com/styled-components/styled-components/blob/a808f40156e62db5c92b06d362e200e5c172a897/packages/styled-components/src/models/StyleSheetManager.js#L33
export default function StyleSheetManager(props: Props) {
  const [plugins, setPlugins] = useState(props.stylisPlugins);
  const contextStyleSheet = useStyleSheet();

  const styleSheet = useMemo(() => {
    let sheet = contextStyleSheet;

    if (props.sheet) {
      // eslint-disable-next-line prefer-destructuring
      sheet = props.sheet;
    } else if (props.target) {
      sheet = sheet.reconstructWithOptions({ target: props.target }, false);
    }

    if (props.disableCSSOMInjection) {
      sheet = sheet.reconstructWithOptions({ useCSSOMInjection: false });
    }

    return sheet;
  }, [props.disableCSSOMInjection, props.sheet, props.target]);

  const stylis = useMemo(
    () =>
      createStylisInstance({
        options: { prefix: !props.disableVendorPrefixes },
        plugins,
      }),
    [props.disableVendorPrefixes, plugins]
  );

  useEffect(() => {
    if (!shallowequal(plugins, props.stylisPlugins))
      setPlugins(props.stylisPlugins);
  }, [props.stylisPlugins]);

  return (
    <StyleSheetContext.Provider value={styleSheet}>
      <StylisContext.Provider value={stylis}>
        {process.env.NODE_ENV !== "production"
          ? React.Children.only(props.children)
          : props.children}
      </StylisContext.Provider>
    </StyleSheetContext.Provider>
  );
}
```

那么根据 StyleSheetManager 中的代码，context 的值是由 styleSheet 提供的，而 styleSheet 的值有以下 4 个选项：

- 默认使用 useStyleSheet 中的返回值
- 存在 sheet 属性使用该 sheet
- 存在 target 重新使用该 target 构建 sheet，并使用该 sheet
- 存在 disableCSSOMInjection 则不使用 CSSOMTag 构建 sheet，并使用该 sheet

因此，只要在组件外部包裹 StyleSheetManager 并使用指定具体的 target，就能在子应用切换后，React 组件重新进行生命周期挂载，生成新的 sheet，之后就可以正常使用 styled-components 了。
