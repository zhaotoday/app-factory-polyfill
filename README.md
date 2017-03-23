## 项目背景
近期，前端团队在开发网龙 99 游手机端上的一个混合应用，叫：群等级，基于应用工厂 JS Bridge 提供的原生能力。项目开发完成后，PC 端提出需要复用我们的 H5 页面（ps：目前 99 游 PC 端默认用 webkit 内核浏览器来渲染 H5 页面）。

## 解决方案
添加 JS Bridge polyfill，模拟 JS Bridge 注入的原生能力，其中无法模拟的 API（如设备相关接口等）方法返回空。

## UC 鉴权
浏览器访问 H5 页面时，无法通过 JS Bridge 的 sdp.restDao 下的 restDao API 进行接口请求授权。这时，PC 端需在 H5 访问地址上带上票据，即：
```js
token=Base64.encode(`${mac}\r\n${access_token}\r\n${nonce}`)
```
UC 鉴权：
```js
new UCModel()
  .addPath(`${access_token}/actions/valid`)
  .POST({
    data: {
      mac: mac,
      nonce: nonce,
      http_method: 'GET',
      request_uri: `/?gid=${store.url.groupId}`,
      host: ENV.host
    }
  })
  .then((res) => {
    store.userId.set(res['user_id'])
    store.token.set(res)
    this.context.router.push('grade')
  })
  .catch(() => {
    this.setState({
      message: '鉴权失败。'
    })
  })
```
其中：
```
request_uri: `/?gid=${store.url.groupId}`
```
需要和 PC 协商，两端算法保持一致。鉴权成功后，将返回的 token 等保存在本地，用于计算接口请求所需的 Authorization 头信息。

## 代码示例

可模拟的方法：
```js
import axios from 'axios'
import json from 'json-bigint'
import auth from './utils/auth'

const PROTECTION_PREFIX = /^\)\]\}',?\n/

/**
 * RESTDao class
 */
class RESTDao {
  constructor() {
    const methods = ['get', 'post', 'patch']

    methods.forEach((value) => {
      this[value] = options => this._request(options.rest)
    })
  }

  /**
   * 接口请求
   */
  _request({authed, baseURL = '', url = '', data = {}, headers = {}, method = 'GET'}) {
    if (authed)
      headers.Authorization = auth.getAuth(method, url, baseURL.split('://')[1])

    return axios({
      responseType: 'text',
      transformResponse: [function (responseText) {
        let data = responseText.replace(PROTECTION_PREFIX, '')
        try {
          data = json.parse(data)
        } catch (e) {
        }
        return data
      }],
      method,
      baseURL,
      url,
      data,
      headers
    })
  }
}

export default window.Bridge ? window.Bridge.require('sdp.restDao').promise() : new RESTDao()
```

不可模拟的方法：
```js
import log from './utils/log'
import promise from './utils/promise'

export default window.Bridge ? window.Bridge.require('sdp.appfactory').promise() : {
  registerWebviewMenu() {
    log('sdp.appfactory', 'registerWebviewMenu', arguments)
    return promise
  },
  unRegisterWebviewMenu() {
    log('sdp.appfactory', 'unRegisterWebviewMenu', arguments)
    return promise
  },
  setMenuVisible() {
    log('sdp.appfactory', 'setMenuVisible', arguments)
    return promise
  }
}