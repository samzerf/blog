---
title: 实现一个自动上传资源到CDN的Webpack插件
date: 2021-03-11 11:07:16
tags:
---

> 导语：日常开发中我们通常会把前端的静态资源上传到CDN，以提高访问速度。本文将会带领大家开发一个Webpack插件，实现把资源打包成zip，然后上传到COS自动解压的功能。

#### 需求分析
在开发一个功能之前，我们先来拆解一下需求。经过分析，其实需要实现的逻辑就是以下几点：
- 在`Webpack`构建完成之前，遍历静态资源，生成zip
- 把zip上传到`COS（CDN）`
- 上传到`COS`后自动解压
因为我们上传的是一个`.zip`的文件，所以在上传完成之后，需要对文件进行解压。这里会以`COS触发`的方式触发一个云函数，用以处理zip文件的解压。

#### Webpack插件的基本结构
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

#### `compiler`和`compilation`
如上面的代码所示，在开发`Webpack`插件时，必然会使用到`compiler`和`compilation`这两个对象。
`compiler`包含了我们此次构建所有的配置信息，同时也提供了一些生命周期钩子，让我们可以在特定的时机执行相应的逻辑。
`compilation`可以理解为此次运行打包的上下文，所有打包过程中产生的结果，都会放到这个对象中。它包含了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息等。当`Webpack`以开发模式运行时，每当检测到一个变化，一次新的`compilation`将被创建。与`compiler`一样，compilation对象也提供了很多事件回调供插件做扩展。
在`UploadToCDN`插件的开发中，我们主要用到了`compiler`的两个钩子`emit`和`afterEmit`。
`emit`会在输出`asset`到`output`目录之前执行，所以我们可以在这个钩子函数上对生成的静态资源进行打包压缩。而`afterEmit`会在输出`asset`到`output`目录之后执行，此时zip文件已经生成，可以进行上传操作。

#### 插件核心代码实现
```javascript
 class UploadToCDN {
    constructor(options) {
      this.options = options
    }
    apply(compiler) {
      compiler.hooks.emit.tapAsync(PLUGIN_NAME, (compilation, callback) => {
        const folder = zip.folder(this.options.filename)
        for (let filename in compilation.assets) {
          const source = compilation.assets[filename].source()
          folder.file(filename, source)
        }
        zip.generateAsync({ type: 'nodebuffer' }).then((content) => {
          this.outputPath = path.join(compilation.options.output.path, `${this.options.filename}.zip`)
          const outputRelativePath = path.relative(compilation.options.output.path, this.outputPath)
          compilation.assets[outputRelativePath] = new RawSource(content)
          callback()
        })
      })

      compiler.hooks.afterEmit.tapAsync(PLUGIN_NAME, (compilation, callback) => {
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
            console.err(err)
          }
          console.log('Upload to cdn success...!', data)
          callback()
        })
      })
    }
  }
```