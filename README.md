# 优化建议

如果数据（列表、树状）很多特别卡的时候，可参考：
列表参考：npm包react-window，https://react-window.vercel.app/#/examples/list/fixed-size
树状参考：npm包react-vtree，https://lodin.github.io/react-vtree/index.html?path=/story/tree--fixedsizetree

## 前端
### 一. 打包优化

1. 提取第三方库（利用webpack的splitChunks，下方有代码demo）。各种第三方的库，变化很少，我们完全可以单独打包，并设置强缓存（expires、cache-control）和协商缓存（lastmodify、Etag）。当项目迭代时，不会再将第三方库打包进包里面了，可以减小包的大小，且配合缓存，即可增加打包速度及页面的加载速度。拆分包后势必会增加http请求，我们可以将资源上传至不同域名或cdn，增加浏览器的并行下载能力，或者使用http2，多路复用
	- 针对低频率基本不变化的文件，并且所有页面都需要用的包（如react、ant），单独打包，并且配置固定版本号
	- 针对中频率变化很少的文件，部分页面才用的包（如lodash，echarts等），单独打包，并且配置固定版本号
	- 针对高频率变化的文件，基本是业务代码，单独打包，并且配置动态版本号
	- 针对其他npm动态加载的包，但是项目中没用到的包，单独打包，并且不引入页面，减小打包文件大小

2. 去除没用到的npm包
	- 通过工具分析（利用npm包depcheck），找出没用到的npm包，进行删除处理
	- 使用Tree Shaking

3. 启用gzip压缩
	- 通过webpack配置，将能在构建时压缩的文件（如css、js等，图片不能gzip压缩），构建时进行压缩（压缩等级为1-9，我们设置为9），避免服务器动态实时压缩，消耗服务器的cpu资源，而且也能节省服务器压缩所需要的时间
	- 压缩好文件后，通过Nginx配置gzip_static，如果已经压缩文件(比如.gz结尾的文件)，服务器直接返回压缩文件至浏览器即可；同时配置gzip，对一些构建时无法压缩的文件，进行动态压缩
	- 入口html不是直接引人.gz文件，而是会引入和.gz文件一样文件名的文件

4. 其他
	- 固定package.js中npm包的版本。避免由于第三方库的升级而导致bug
	- 修改webpack配置，使得打包时不生成sourceMap和License文件，优化打包时间及包大小
	- 域名预解析
	- 很少变更的数据做localstorage缓存，比如城市编码、行业分类等
	- 通过配置，能够展示包的构成情况（webpack-bundle-analyzer），将公用的包提出，将无用的包删除；
	- 明细化的展示各个包的打包时间（speed-measure-webpack-plugin），然后进行响应的优化（比如cacheDirectory），优化打包速度
	- 通过配置，能够将开发环境构建时的文件写入硬盘（默认是写入内存的），方便查看优化
	- babel配置增加@babel/transform-runtime、["@babel/preset-env", {modules: false}]、"compact": false
	- webpack的optimization增加usedExports: true

5. Tree-shaking。参考：https://zhuanlan.zhihu.com/p/417686391；参考：https://blog.csdn.net/weixin_45047039/article/details/110384743
6. 打包速度优化参考：[https://www.cnblogs.com/imwtr/p/7801973.html](https://www.cnblogs.com/imwtr/p/7801973.html)
7. webpack5可以更好的打包。比如optimization.usedExports删除未使用的属性等等；

```javascript

	// 开发环境方便源码调试
	devtool: isDev ? 'cheap-module-source-map' : false,
```

```javascript
	
	// 提取第三方库。splitChunks配置参考
	// 如果碰见报错：Cannot read property 'pop' of undefined，只要加上enforce: true就好了，参考echarts
	optimization: {
		...config.optimization,
		splitChunks: {
			// chunks: 'initial',
			cacheGroups: {
				// 第三方不变的包，单独打包插入HTML。当项目迭代时，不会再将第三方库打包进包里面了，可以减少包的大小，提高性能
				'ant-react': {
					chunks: 'all',
					test: /ant|react/,
					name: "ant-react",
					priority: -1,
					reuseExistingChunk: true,
				},
				echarts: {
					chunks: 'all',
					test: /echarts/,
					name: "echarts",
					priority: -1,
					reuseExistingChunk: true,
					enforce: true
				},

				// 以下是使用不到的包(比如其他npm包内部依赖的包)，将其单独打包，避免打包进入bundle.js
				'monaco-editor': {
					chunks: 'all',
					test: /monaco-editor/,
					name: "monaco-editor",
					priority: -1,
					reuseExistingChunk: true,
				},

				// 以下是可能变化的包
				nodeModules: {
					// 将node_modules打包在一起
					chunks: 'all',
					test: /[\\/]node_modules[\\/]/,
					priority: -3,
					// reuseExistingChunk: true,
					name: 'nodeModules',
				},
				qikecommon: {
					// 将其他代码打包在一起
					chunks: 'all',
					priority: -4,
					enforce: true,
					// reuseExistingChunk: true,
					name: 'qikecommon'
				}
			}
		}
	}
	
```
```javascript
	
	// webpack-bundle-analyzer配置参考。注意判断环境，debug环境才配置这个
	plugins: [
		...config.plugins,
		new BundleAnalyzerPlugin({
			analyzerMode: 'server',
			analyzerHost: '127.0.0.1',
			analyzerPort: 8889,
			reportFilename: 'report.html',
			defaultSizes: 'parsed',
			openAnalyzer: true,
			generateStatsFile: false,
			statsFilename: 'stats.json',
			statsOptions: null,
			logLevel: 'info'
		}),
	]

```
```javascript

	// webpack的gzip压缩。注意判断环境，生产环境才配置这个
	const CompressionPlugin = require("compression-webpack-plugin");
	plugins: [
		...config.plugins,
		new CompressionPlugin({
			test: /\.(js|css|html|svg)$/i,
			// 使用不到的包，禁用压缩，节省时间
			exclude: /monaco\-editor|tinymce|exceljs/,
			algorithm: 'gzip',
			compressionOptions: { level: 9 },
			// minRatio: 0.8,
			filename: '[file].gz[query]'
		}),
	]

```
```javascript
	
	// 删除CSS的sourcemaps。开发环境和生成环境都需要
	const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
	webpackConfig.optimization.minimizer.push(new CssMinimizerPlugin({
			sourceMap: false,
			minimizerOptions: {
				sourceMap: false
			},
			minify: async (data, inputMap, minimizerOptions) => {
				// eslint-disable-next-line global-require
				const CleanCSS = require('clean-css');

				const [
					[filename, input]
				] = Object.entries(data);
				const minifiedCss = await new CleanCSS({
					sourceMap: false
				}).minify({
					[filename]: {
						styles: input,
						sourceMap: inputMap,
					},
				});

				return {
					css: minifiedCss.styles,
					map: minifiedCss.sourceMap ? minifiedCss.sourceMap.toJSON() : '',
					warnings: minifiedCss.warnings,
				};
			},
		}));
```

```javascript

	// 删除license文件。开发环境和生成环境都需要
	webpackConfig.optimization.minimizer.push(new TerserPlugin({
		terserOptions: {
			parse: {
				ecma: 8,
			},
			compress: {
				ecma: 5,
				warnings: false,
				comparisons: false,
				inline: 2,
			},
			mangle: {
				safari10: true,
			},
			output: {
				ecma: 5,
				comments: false,
				ascii_only: true,
			},
		},
		// 不打包LICENSE文件
		extractComments: false,
		sourceMap: false
	}))
```

### 二、页面优化
1. 为了减少页面白屏时间，须JS放页面后端，也要注意JS处理事件效率等，减少阻塞。资源延迟加载等提高首屏展示时间
2. 减少HTTP请求数。请求数多时，为了提高浏览器的并行能力，可采用多域名及cdn，配置域名预解析
3. 图片选择合适的格式（如webp，注意优雅降级）、大小及质量，icon可以通过雪碧图、font icon及base64等方式
4. 压缩静态资源。对于敏感资源，可以采用混淆压缩
5. 页面布局应语义化，也应避免标签、css层级太深
6. 善于使用本地存储（localStorage、sessionStorage）来缓存一些配置数据，比如国家、省份等信息
7. 可以通过performance.timing和performance.getEntries()查看页面资源的请求情况
8. 写代码时，时刻注意闭包、重绘、重排等，善于启用GPU，善于使用requestAnimationFrame、createDocumentFragment等等
9. 页面静态化
10. 更多建议请参考pagespeed或者yslow

## 后端
1. 尽量减少转发代理等次数
2. 对资源设置Content-Length属性，可以让浏览器提前知道数据的多少，而无需自己去实时检测数据是否传输完毕，提高其效率。
3. 对实时性要求不高的数据做缓存
4. 尽量瘦身接口返回内容，避免将一些用不到的数据也返回前端，导致传输数据量太大
5. 增加Server-Timing响应标头，方便我们了解请求的数据各个阶段所花费的时间，比如db时间等
6. 接口缓慢原因，是否因为配置太低，太多计算耗费CPU，导致性能降低，考虑更换更高性能的服务器

## 运维
1. 可以采用http2协议。多路复用、头部压缩等等优势
2. 对静态资源启用gzip压缩，压缩等级看需求而定，等级越高压缩比例越大，但是也越耗CPU。注意图片别gzip压缩
3. 对于常年不变或变化很少的资源，启用强制缓存（expires、cache-control），不请求网络；其他资源可以启用协商缓存，尽量使用lastModify，304返回头部，不返回body。尽量少用eTag，比较耗CPU，一定要etag时，尽量部分内容etag，避免全量etag
4. 对于重要项目或者pv、uv高项目，采用负载均衡

## 网络
1. CDN
2. dns缓存
3. 域名预解析
4. 负载均衡
