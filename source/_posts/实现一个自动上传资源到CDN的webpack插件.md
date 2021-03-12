---
title: 实现一个自动上传资源到CDN的Webpack插件
date: 2021-03-11 11:07:16
tags:
---

> 导语：日常开发中我们通常会把前端的静态资源上传到CDN，以提高访问速度。本文将会带领大家开发一个Webpack插件，实现把资源打包成zip，然后上传到COS自动解压的功能。
<!-- more -->
### 需求分析
在开发一个功能之前，我们先来拆解一下需求。经过分析，其实需要实现的逻辑就是以下几点：
- 在`Webpack`构建完成之前，遍历静态资源，生成zip
- 把zip上传到`COS（CDN）`
- 上传到`COS`后自动解压
因为我们上传的是一个`.zip`的文件，所以在上传完成之后，需要对文件进行解压。这里会以`COS触发`的方式触发一个云函数，用以处理zip文件的解压。

### Webpack插件的基本结构
一个`Webpack`插件其实就是一个构造函数，原型上定义了一个`apply`方法，这个方法会在Webpack启动时被调用。所以可以在`apply`方法中设置钩子函数，在特定的时机执行指定的逻辑。
下面就是一个`Webpack`插件的基本结构：
```javascript
  class UploadToCDN {
    apply (compiler) {
      compiler.hooks.somehook.tap('plugin-name', (compilation) => {
        // to do something
      })
    }
  }
```

### `compiler`和`compilation`
如上面的代码所示，在开发`Webpack`插件时，必然会使用到`compiler`和`compilation`这两个对象。
`compiler`包含了我们此次构建所有的配置信息，同时也提供了一些生命周期钩子，让我们可以在特定的时机执行相应的逻辑。
`compilation`可以理解为此次运行打包的上下文，所有打包过程中产生的结果，都会放到这个对象中。它包含了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息等。当`Webpack`以开发模式运行时，每当检测到一个变化，一次新的`compilation`将被创建。与`compiler`一样，compilation对象也提供了很多事件回调供插件做扩展。
在`UploadToCDN`插件的开发中，我们主要用到了`compiler`的两个钩子`emit`和`afterEmit`。
`emit`会在输出`asset`到`output`目录之前执行，所以我们可以在这个钩子函数上对生成的静态资源进行打包压缩。而`afterEmit`会在输出`asset`到`output`目录之后执行，此时zip文件已经生成，可以进行上传操作。

所以我们的插件大概是这样子的：

```javascript
  class UploadToCDN {
    constructor(options) {
      this.options = options
    }
    apply(compiler) {
      // 因为会有异步操作，所以这里用tapAsync， 当逻辑执行完成时调用callback通知Webpack 
      compiler.hooks.emit.tapAsync(PLUGIN_NAME, (compilation, callback) => {
        // 在资源输出到dist之前进行压缩
      })
      compiler.hooks.afterEmit.tapAsync(PLUGIN_NAME, (compilation, callback) => {
        // 在资源输出后，把dist上传到cos
      })
    }
  }
```

### 插件核心代码实现
- **压缩资源文件**
`compilation.assets`这个对象上会有我们本次构建的资源，所以我们可以通过遍历获取生成的静态资源，同时也可以通过给`compilation.assets[filename]`赋值输出内容。这里我们会使用`JSZip`对文件进行压缩。
```javascript
  compiler.hooks.emit.tapAsync(PLUGIN_NAME, (compilation, callback) => {
    const folder = zip.folder(this.options.filename)
    // 遍历资源，把资源放进zip包中
    for (let filename in compilation.assets) {
      const source = compilation.assets[filename].source()
      folder.file(filename, source)
    }
    zip.generateAsync({ type: 'nodebuffer' }).then((content) => {
      // 生成zip文件，并把文件输出到指定目录
      this.outputPath = path.join(compilation.options.output.path, `${this.options.filename}.zip`)
      const outputRelativePath = path.relative(compilation.options.output.path, this.outputPath)
      compilation.assets[outputRelativePath] = new RawSource(content)
      callback() // 通知Webpack该钩子函数已经执行完毕
    })
  })
```

- **上传zip文件**
因为我使用的是腾讯云的对象存储，所以这里直接通过腾讯云的`SDK`把压缩包上传到指定的`Bucket`。
```javascript
  compiler.hooks.afterEmit.tapAsync('UploadToCDN', (compilation, callback) => {
    cos.putObject({
      Bucket: 'your bucket', 
      Region: 'region', 
      Key: `${this.options.filename}.zip`,
      StorageClass: 'STANDARD',
      Body: fs.createReadStream(this.outputPath),
      onProgress: progressData => {
        console.log(JSON.stringify(progressData))
      }
    }, (err, data) => {
      if (err) {
        console.error('Upload to cdn fail.......!')
        console.err(err)
      }
      console.log('Upload to cdn success...!', data)
      callback()
    })
  })
```

### 配置云函数
上面我们上传的是一个`zip`文件，上传之后我们还需要对`zip`包进行解压，让各个静态资源处于正确的访问路径上。
具体做法是：
1. 创建两个`Bucket`(`static`和`upload`)。`static`用于存放我们真实会访问到的静态资源，而`upload`则专门存储我们刚刚上传的`zip`文件。
2. 选择模板创建云函数，函数模板选择Nodejs解压zip包。
![](https://samzherf-1256415834.file.myqcloud.com/picbed/cf.png)
![](https://samzherf-1256415834.file.myqcloud.com/picbed/cf_tpl.png)
3. 设置云函数为`COS触发`
![](https://samzherf-1256415834.file.myqcloud.com/picbed/cos_emitter.png)
关于云函数的代码这里就不贴出来了，因为都是基于模板生成的，基本不用自己手写代码。
**解压云函数的逻辑**
当有zip文件上传到`upload Bucket`时，云函数会被触发，同时云函数能拿到zip文件的下载地址。之后调用SDK的API下载zip，下载完成之后进行解压，然后遍历目录，把资源上传到另一个`Bucket static`上。经过这些处理之后，我们就能在`static`目录上访问到我们刚刚打包上传的`js`, `html`, `css`, `image`等等。

经过以上一系列逻辑实现之后，上传CDN插件总算完成了。具体源码请查看：https://github.com/samzerf/upload-to-cos