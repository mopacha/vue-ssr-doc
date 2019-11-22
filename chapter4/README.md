
# 常见问题

## mini-css-extract-plugin在SSR上不能用

原因：这个插件会用document方法，往模板插入标签。服务器渲染跑的是node.js环境里没有document这个方法，就会报错：document is not defined。

解决方案：

`webpack.server.config.js` 端屏蔽

```js
//屏蔽方法
class ServerMiniCssExtractPlugin extends MiniCssExtractPlugin {
  getCssChunkObject(mainChunk) {
    return {};
  }
}
```

`webpack.client.config.js`端 正常使用mini-css-extract-plugin


## 集群模式PM2+Log4js 写入Log失败

1. log4js的配置中增加这两个配置

```js
{
    pm2: true,
    pm2InstanceVar: "INSTANCE_ID" // 与pm2的配置对应
}
```
2. 安装pm2-intercom;

```bash
pm2 install pm2-intercom
```


## element-ui的组件 el-table 不支持服务端渲染

github 上面 elment-ui 已经提issue，还未修复。暂不影响业务功能


##  svg-sprite-loader 在server端转换不成功

**1. 在`entry-server.js`中把symbolContent 放在 `context`中**

```js
//entry-server.js
import getSvgContent from '@/util/svg-spirate'

export default (context) => {
	return new Promise((resolve, reject) => {
......
		context.svgContent = getSvgContent() // 把svg-symbol 交给server-render
......
}

```


```js
// @/util/svg-spirate
export default () => {
	const req = require.context('@/icons/svg', false, /\.svg$/)
	const requireAll = requireContext => requireContext.keys().map(i => requireContext(i))
	const svgContent = requireAll(req)

	let symbolContent = ''

	svgContent.map(item => {
		symbolContent = symbolContent + item.default.content
	})

	const pageContent =
		`<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" style="position: absolute; width: 0; height: 0" id="__SVG_SPRITE_NODE__">
			${symbolContent}
		</svg>`

	return pageContent
}

```
**2. 在server端将 `svgContent`注入到html中**

```js
//renderer.renderToString 开始执行行entry-erver.js
	renderer.renderToString(context, (err, html) => {
		if (err) {
			ctx.log.debug(`renderToString:`, err)
			return reject(err)
		}
		const svgContent = context.svgContent
		const $ = cheerio.load(html)
		$('body').prepend(svgContent)
		resolve($.html())
	})
```

## 关于storage、document、window的使用

>在vue的生命周期函数中，beforeCreate和created会在服务端和客户端执行，其他钩子都在客户端执行，所以，如果在beforeCreate和created，或是直接在vuex和router入口文件中使用了storage、document以及window这些在浏览器端js才有的属性或对象时，就会报错。为了避免这个问题，应该在客户端渲染的钩子中执行。