---
author: Zhou Yu
pubDatetime: 2023-10-16T06:36:02.647Z
title: Custom next.js 500 page, try to expose internal error.
postSlug: custom-nextjs-500-page
featured: true
tags:
  - next.js
description: "Custom next.js 500 page, try to expose interal error."
---

When we use Next.js, typically, we can't do much customization on the 500 error page besides changing the styles. If you want to display the specific error code thrown by an API to facilitate user bug feedback, what can you do?

## Table of contents

## 解决办法

```ts
const Error = ({ statusCode }: ErrorProps) => {
  // hack
  const [errCode, setErrCode] = useState('')
  useEffect(() => {
    try {
      const p = JSON.parse(
        document.getElementById('__NEXT_DATA__')?.textContent as string
      )
      setTraceid(p.props.pageProps.errCode)
    } catch (err) {}
  }, [])

  return (
    <>
      <h1>500</h1>
      <div>{errCode}</div>
    </>
  )
}

Error.getInitialProps = async ({
  res,
  err,
  pathname,
  query,
  AppTree,
}: NextPageContext) => {
  const errorInitialProps = await NextError.getInitialProps({
    res,
    err,
    pathname,
    query,
    AppTree,
  })

  let errCode = err.code
  return {
    ...errorInitialProps,
    errCode,
  }
}

export default Error
```

## The code is easy to understand, but why do it this way?

`getInitialProps` 返回的 `errCode` 是会丢失的。直接用`props`去接收`errCode`你就会发现页面上的`errCode`闪烁一下就消失了。

_假设如果用`props`去接收`errCode`会发生什么？_

next.js渲染是分为两部分的，首先是服务端渲染出html显示到页面上, 然后客户端再次渲染出html替换掉服务端渲染的html（这个过程叫做hydrate）。

首先服务端渲染出的结果是没有问题的，但是客户端hydrate的过程中渲染出来的结果是没有`errCode`的导致了上述现象的发生

## 为什么客户端渲染的时`props.errCode`丢失了

我的的`errCode`是从`err.code`取出来的

- 服务端调用`getInitialProps({err: err})`时用的是代码里throw出的原始错误
- 客户端调用`getInitialProps({err: err})`时用的错误是

```json
{
    message: "500 - Internal Server Error."
    name: "Internal Server Error."
    statusCode :500
  }
```

所以客户端渲染时 `errCode` 返回的就是`undefined`，然后就从页面消失了。

## 为什么这个错误变掉了？

直接上next.js源码，nextjs在序列化`error`给客户端用时调用了以下函数

```
function serializeError(
  dev: boolean | undefined,
  err: Error
): Error & {
  statusCode?: number
  source?: typeof COMPILER_NAMES.server | typeof COMPILER_NAMES.edgeServer
} {
  if (dev) {
    return errorToJSON(err)
  }

  return {
    name: 'Internal Server Error.',
    message: '500 - Internal Server Error.',
    statusCode: 500,
  }
}
```

这里原始错误就消失了

## 为什么next.js 这么设计

这种情况可能是由于 Next.js 默认的错误处理机制以及安全性考虑而导致的。Next.js的500错误页面通常设计成非常简洁和最小化，以减少潜在的信息泄漏风险，这可以增加应用程序的安全性。

---

到此已经全部明了，我们只要拿到next.js服务端返回的原始数据就可以解决最开始的问题了。

> next.js服务端返回的原始数据怎么拿呢

```ts
const p = JSON.parse(
  document.getElementById("__NEXT_DATA__")?.textContent as string
);
```

或者你直接用

```ts
window.__NEXT_DATA__;
```

也行

嘿嘿！
