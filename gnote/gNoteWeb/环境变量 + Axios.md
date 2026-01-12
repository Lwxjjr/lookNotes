# 环境变量
.env.development:
```SH
VITE_APP_ID=123456
VITE_SERVER_URL= http://127.0.0.1:8080
```

vite.config.ts:
```TS
import { fileURLToPath, URL } from "node:url";
import { defineConfig, loadEnv } from "vite";
import vue from "@vitejs/plugin-vue";

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd());
  console.log(env.VITE_APP_ID);
  return defineConfig({
    plugins: [vue()],
    envDir: "./",
    resolve: {
      alias: {
        "@": fileURLToPath(new URL("./src", import.meta.url)),
      },
    },
    // 解决跨域问题
    server: {
      host: "0.0.0.0",
      port: 5173,
      proxy: {
        "/api": {
          target: env.VITE_SERVER_URL,
          changeOrigin: true,
        },
      },
    },
  });
});

```
这表示 Vite 会从项目的根目录（`./`）开始查找环境变量文件。例如，Vite 会查找以下文件：
- `.env`：适用于所有环境 
- `.env.development`：仅适用于开发环境 
- `.env.production`：仅适用于生产环境

---
# Axios
最初的旧浏览器页面在向服务器请求数据时，由于返回的是整个页面的数据，所以整个页面都会强制刷新一下，这对于用户来讲并不是很友好，因为当我们只需要请求部分数据时，服务器返回给我们的确是整个页面的数据，这样会造成网络资源的占用，即十分消耗网络资源。为了提高数据请求效率，异步网络请求Ajax就应运而生了，它可以在页面无刷新的情况下请求数据。因此，这样一来，==当我们只是需要修改页面的部分数据时，可以实现不刷新页面的功能==。  
AJAX = Asynchronous JavaScript and XML（异步的 JavaScript 和 XML）  
—— 异步网络请求 —— Ajax能够让页面无刷新的请求数据 ——  
1.  ==Ajax的实现依赖于XMLHttpRequest对象==，即XMLHttpRequest可以实现Ajax，[Ajax](https://juejin.cn/post/6985505513765568526 "Ajax")是一种技术方案，但并不是一种新技术。它依赖现有的CSS/HTML/JavaScript，而其中最核心的依赖是浏览器提供的XMLHttpRequest对象，是这个对象使得浏览器可以发出HTTP请求与接收HTTP响应
2.  ==Axios在此基础上封装了XMLHttpRequest==，即Axios可以实现Ajax
Axios 是一个基于 promise 的网络请求库  

> Promise: 在编程中是一个用于管理异步操作的工具  
> Promise 就像是给异步操作的一个承诺，当操作完成时会通知你结果，无论成功还是失败  

## 封装Axios
```TS
// src/api/index.ts

import axios from "axios";
import {Message} from "@arco-design/web-vue";

export const useAxios = axios.create({
    // baseURL: "",
})

// 列表查询的入参
export interface Params {
    key?: string
    sort?: string
    limit?: number
    page?: number
}

// 普通的响应类型
export interface Response<T>{
    code: number
    data: T
    msg: string
}

// 列表查询页的响应类型
export interface ListResponse<T>{
    code: number
    data: {
        count: number
        list: T[]
    }
    msg: string
}


// 请求中间件
useAxios.interceptors.request.use((config) => {
    config.headers["token"] = "1234"
    return config
})


// 响应中间件
useAxios.interceptors.response.use((response) => {
    if (response.status === 200) {
        return response.data
    }
    // 如果接口错误，要进行错误信息提示
    console.log("接口错误", response.statusText)
    Message.error(response.statusText)
    return Promise.reject(response.statusText)
    return response.data
}, (error) => {
    Message.error(error.message)
    console.log("服务错误", error)
    return Promise.reject(error)
})
```

## 用户登录接口
```TS
// src/api/user_api.ts

import {useAxios} from "@/api/index";
import type {Response} from "@/api/index";


export interface LoginRequest {
    userName :string
    password: string
}

// 登录成功之后，直接返回token
export function loginApi(data: LoginRequest): Promise<Response<string>> {
    return useAxios.post("/api/login", data)
}
```
