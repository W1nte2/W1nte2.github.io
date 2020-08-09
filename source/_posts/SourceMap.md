---
layout: update
title:  "记一次SourceMap前端源码泄露"
date:   2020-01-01 12:37:44 +0800
categories: jekyll update
---


# 记一次SourceMap前端源码泄露
前后端分离技术已成为互联网项目开发的业界标准使用方式。其中就有利用Webpack技术将JS类拓展语言进行打包发布。<br>
虽然前后端分离给项目开发带来了许多便利，但这样后端接口会完全暴露在JS文件中；另外，如果生成了SourceMap文件且未及时清除该文件，那么项目外人员可利用该文件还原网站前端源代码。
主流浏览器都自带解析Source Map文件功能

### Tips!
> 如果不想让vue源文件显示出来，可以在`config/index.js`中`build`下的 `productionSourceMap: true, `改为`productionSourceMap: false,`即可

### 0x00 安装restore-source-tree

```shell
git clone https://github.com/laysent/restore-source-tree
cd restore-source-tree
npm install
```
发现报错<br>
![error](/assets/SourceMap/npm_error.png)
>分析：<br>
因为是win10 环境，但是`NODE_ENV`为unix/linux的环境变量，所以需要安装cross-env包,并修改环境变量。

```npm install -g cross-env```

然后修改目录下package.json文件中的配置
`vim package.json` 将package.json文件中的`"build": "NODE_ENV=production babel index.js --out-file dist.js",`替换为：`"build": "cross-env NODE_ENV=production babel index.js --out-file dist.js",`

修改后package.json如下：
```json
{
  "name": "restore-source-tree",
  "version": "0.1.1",
  "description": "Restores file structure from source map",
  "main": "dist.js",
  "bin": {
    "restore-source-tree": "bin/restore-source-tree.js"
  },
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "cross-env NODE_ENV=production babel index.js --out-file dist.js",
    "prepublish": "npm run build"
  },
  "author": "Alexander <alexkuz@gmail.com> (http://kuzya.org/)",
  "license": "MIT",
  "dependencies": {
    "commander": "^2.9.0",
    "glob": "^7.1.3",
    "minimist": "^1.2.0",
    "mkdirp": "^0.5.1",
    "source-map": "^0.5.6"
  },
  "devDependencies": {
    "babel-cli": "^6.11.4",
    "babel-eslint": "^6.1.2",
    "babel-preset-es2015": "^6.9.0",
    "babel-preset-stage-0": "^6.5.0",
    "eslint": "^3.1.1"
  }
}
```

解决了系统环境不同导致的报错问题后，继续执行: 

```npm install```

```shell
audited 410 packages in 15.484s

3 packages are looking for funding
  run `npm fund` for details

found 1 low severity vulnerability
  run `npm audit fix` to fix them, or `npm audit` for details
```
最终会得到如上提示，编译成功！<br>
但是因为Windows平台不允许路径、文件中含有?等特殊字符，所以在使用restore-source-tree还原代码时会出现以下报错:
```
$ restore-source-tree main-3dba0acc5bd634c8d86e.js.map
Processed 1498 files
Failed writing file output\src\styles\login.less?b295
```
继续修改`vim dist.js`:在文件95行处增加`outPath=outPath.replace('?','_');`。
```js
  (0, _mkdirp2.default)(dir, function (err) {
    if (err) {
      console.error('Failed creating directory', dir);
      process.exit(1);
    } else {
	  outPath=outPath.replace('?','_');//替换特殊字符
      _fs2.default.writeFile(outPath, content, function (err) {
        if (err) {
          console.error('Failed writing file', outPath);
          process.exit(1);
        }
      });
    }
  }
```
### 0x01 还原代码
继续使用 restore-source-tree还原代码是没有再出现以上报错！
```
$ restore-source-tree main-3dba0acc5bd634c8d86e.js.map
Processed 1498 files
```
成功还原代码：<br>
![success](/assets/SourceMap/restore-source-tree_success.png)
