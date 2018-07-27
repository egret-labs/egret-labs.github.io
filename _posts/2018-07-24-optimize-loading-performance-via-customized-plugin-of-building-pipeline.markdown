---
layout: post
title:  "使用白鹭引擎构建管线自定义插件优化游戏加载效率"
date:   2018-07-24 14:00:00 +0800
categories: 白鹭引擎

excerpt_separator: <!--more-->
---

开发者在制作完一款 HTML5 游戏后，无论是发布到 Web 平台，或者是微信小游戏或者微端等其他方式，都需要不断改进游戏的加载效率。这样做一方面可以提升游戏体验，提高用户留存，也可以帮助用户节省流量，帮助开发者自身降低服务器带宽费用，可谓是一举多得。

在诸多优化方式之中，投入产出比比较大的就是将 HTTP 请求进行合并。



<!--more-->



# HTTP 请求的技术原理

一个HTTP请求的主要过程是：

* DNS解析 ( DNS Lookup ) 
* 建立TCP连接 ( Initial connection ) 
* 发送请求 ( Request sent ) 
* 等待服务器返回首字节 ( Waitting TTFB ) 
* 接收数据 ( Conent Download )

我们可以从 Chrome Devtools 中看到这其中的每一个步骤消耗的时间

![HTTP请求的主要过程]({{ "/assets/2018-07-24-optimize-loading-performance-via-customized-plugin-of-building-pipeline-1.jpg" | absolute_url }})


通过上图我们可以分析出，加载 100 个 1k 的资源的效率远远低于加载 1个 100k 资源的效率。因此为了优化游戏加载效率，我们需要将游戏资源进行合并，将多个 HTTP 请求优化为一个请求。


# 核心思路

在一款 HTML5 游戏中，游戏资源主要分为如下几类：

* JavaScript 代码：本文档不考虑此类型
* 声音资源：本文档不考虑此类型
* 图片资源：可以使用 TextureMerger 工具将图片合并为纹理集。
* 文本及二进制配置文件：我们以 JSON 配置文件合并为例，重点介绍如何利用白鹭引擎构建管线的自定义插件进行合并


# 困难

将多个配置文件合并为一个的话，会带来如下两个问题：

* 很多配置文件需要频繁修改，每次修改完配置文件，就必须再次对资源进行合并
* 配置文件合并之后，必须对应修改加载逻辑，否则仍然会加载合并前的配置文件

为了解决上述两个问题，我们意识到，需要设计如下两个功能：

* 提供一个在游戏发布时自动将配置文件进行合并的功能
* 在游戏运行时代码进行尽可能小的调整，封装统一合并前配置与合并后配置的加载调用方式。



# 准备工作

我们首先创建一个新的白鹭游戏项目，并在项目的 resource/config/ 文件夹中添加10个配置文件，分别命名为1.json / 2.json ,并在 Egret Wing 中将这些文件加到 default.res.json 的 preload 组中。然后编写如下代码

```typescript

await RES.loadConfig("default.res.json", "resource/");
await RES.loadGroup("preload", 0, loadingView);
const json = RES.getRes("1_json");
console.log(json)
```
上述代码执行后，观察 Chrome Devtools 的 Network 面板，会发现游戏会加载10个配置文件

![加载10个配置文件]({{ "/assets/2018-07-24-optimize-loading-performance-via-customized-plugin-of-building-pipeline-2.png" | absolute_url }})


准备工作完成后，我们就开始尝试为这个项目的资源加载效率进行优化



## 步骤一：自动合并功能

通过扩展白鹭引擎的构建管线编写一个插件，我们可以很轻松的实现配置文件的自动合并功能。具体代码如下：


在 ```scripts/myplugin.ts``` 中添加：

```typescript
export class MergeJSONPlugin implements plugins.Command {

    private obj = {};

    constructor() {
    }

    async onFile(file: plugins.File) {

        if (file.extname == '.json' // JSON 文件参与合并
            && file.origin.indexOf("resource/config/") == 0) { // 只有 resource/config 文件夹中的文件参与合并

            const url = file.origin.replace("resource/", "");
            const content = file.contents.toString()
            this.obj[url] = JSON.parse(content);
        }
        return file;
    }

    async onFinish(commandContext: plugins.CommandContext) {
        commandContext.createFile('resource/total.json', new Buffer(JSON.stringify(this.obj)));
    }
}
```

在 ```scripts/config.ts```中添加：

```typescript
import { MergeJSONPlugin } from './myplugin'; //引入 MergeJSONPlugin


const config: ResourceManagerConfig = {


    buildConfig: (params) => {

        const { target, command, projectName, version } = params;
        if (command == 'build') {
            const outputDir = '.';
            return {
                outputDir,
                commands: [
                    new ExmlPlugin('debug'), 
                    new IncrementCompilePlugin(),
                    new MergeJSONPlugin(), // 添加 MergeJSONPlugin
                ]
            }
        }
        else if (command == 'publish') {
            // 在这里也需要 添加 MergeJSONPlugin，具体代码略
        }
}
```

上述功能添加完毕后，无论执行 ```egret build``` 还是 ```egret publish```，均会生成一个名为 ```total.json```的文件，其中已经对所有的JSON文件进行了合并。 


## 步骤二：修改运行时代码


生成了合并后的配置文件后，如果运行项目，会发现加载的仍然是合并前的配置文件，这需要我们对项目进行一些调整，具体调整方式如下：

首先，编写一个 JSONProcessor，用于处理 JSON 类型的数据


```typescript
class JSONProcessor implements RES.processor.Processor {

    private totalContent: any;

    async onLoadStart(host: RES.ProcessHost, resource: RES.ResourceInfo): Promise<any> {
        console.log('load json', resource.url)
        if (resource.url.indexOf("config/") == -1) {
            return host.load(resource, RES.processor.JsonProcessor)
        }
        else {
            if (!this.totalContent) {
                const name = "total_json";
                if (!RES.hasRes(name)) {
                    const url = "total.json";
                    const type = 'json';
                    const root = RES.config.config.resourceRoot;
                    const resource = { name, url, type, root };
                    RES.$addResourceData(resource);
                    this.totalContent = await RES.getResAsync(name)
                }
                else {
                    await RES.getResAsync(name)
                }
            }
            return this.totalContent[resource.url];
        }
    }

    onRemoveStart(host: RES.ProcessHost, resource: RES.ResourceInfo): Promise<any> {
        return Promise.resolve();
    }
}
```

之后，在游戏项目的启动入口，使用新的 JSONProcessor 覆盖掉引擎内置的 JSONProcessor，具体代码如下：

```typescript
RES.processor.map("json", new JSONProcessor())
```

修改完这两处代码后，所有的加载逻辑都不会按照引擎默认的方式去加载配置文件，而是从合并后的配置文件中提取对应的部分返回给上层逻辑。


通过这种方式，开发者完全无需调整业务逻辑代码，就实现了加载合并后的配置文件的功能。



# 总结

通过这篇教程，开发者可以了解到：

* 合并 HTTP 请求可以优化加载效率的原理
* 如何将 AssetsManager 和构建管线自定义插件配合使用，合并 HTTP 请求，优化加载速度
* 如何在整个过程中不对具体的业务逻辑进行修改


项目源码您可以在 https://github.com/WanderWang/Egret_MergeJSON 下载