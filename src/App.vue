<script setup>
import axios from 'axios';

// axios 实例
const instance = axios.create({
  baseURL: import.meta.env.VITE_URL
})
let pending = []  // 队列
let isPending = false

const handlerFetch = () => {
  instance.get('/handlerToken_test', { needToken: true })
}
const handlerLogin = () => {
  instance.post('/handlerToken_login')
    .then((response) => {
      if (response.data.code === 20000) {
        localStorage.setItem('token', response.data.data.access_token)
        localStorage.setItem('refresh_token', response.data.data.refresh_token)
      }
    })
    .catch((error) => {
      console.error(error)
    })
  // /login
}
const handlerRefresh = () => {
  instance.post('/handlerToken_refresh')
}

const refreshToken = () => {
  console.log('请求刷新。。。')
  return instance.post('/handlerToken_refresh', {
    refresh_token: localStorage.getItem('refresh_token')
  })
}

// 请求拦截器
instance.interceptors.request.use(
  (config) => {
    // 判断是否需要携带 token 头信息
    if (config.needToken) {
      // 从本地存储中获取 token
      const token = localStorage.getItem('token');
      if (token) {
        // 将 token 头信息添加到请求头中
        config.headers.Authorization = `Bearer ${token}`;
      }
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// 响应拦截器
instance.interceptors.response.use(
  (response) => {
    const { code } = response.data
    // 40100 用户身份验证失败
    if (code === 40100) {
      const originalConfig = response.config; // 保存下请求信息
      // 判断队列
      if (isPending) {
        console.log('已经有刷新请求了，先加到队列等着吧。。。')

        return new Promise((resolve) => {
          pending.push((token) => {
            originalConfig.headers['Authorization'] = token
            console.log('执行队列。。。')
            resolve(instance(originalConfig))
          })
        })
      } else {
        console.log('请求40100了，开始刷新token')
        isPending = true
        return refreshToken()
          .then((response) => {
            console.log('刷新token返回内容', response)
            if (response.data.code === 20000) {
              localStorage.setItem('token', response.data.data.access_token)
              localStorage.setItem('refresh_token', response.data.data.refresh_token)
              // 处理队列
              pending.forEach((cb) => cb(response.data.data.refresh_token))
              // 清空队列
              pending = []
              // 带着新token重新发起当前请求
              originalConfig.headers['Authorization'] = response.data.data.refresh_token
              return instance(originalConfig)
            } else if (response.data.code === 40101) {
              console.log('刷新token也过期了，重新登录吧！')
              return response.data
            }

          })
          .catch((error) => {
            console.error(error)
          })
          .finally(() => {
            isPending = false
          })

      }
    } else {
      return response;
    }

  },
  async (error) => {
    return Promise.reject(error);
  }
);


</script>

<template>
  <div>
    <h1>hello world.!</h1>

    <div>
      <button @click="handlerFetch">fetch</button>
      <button @click="handlerLogin">login</button>
      <button @click="handlerRefresh">refresh</button>
    </div>
  </div>
</template>
