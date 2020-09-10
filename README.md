
<img src="https://github.com/SimonCY/slate-iOS/blob/master/imgs/slate_logo.png?raw=true" alt="图片替换文本" width="180" height="100" align="bottom" />

###### 导语：

###### 本文主要是对slate组件化方案实现的记录（路由、模块划分及模块代码生命周期、模块集成管理方式），并非教学文章，如果你有好的意见或建议欢迎在评论区探讨。

### 前言
- - -

   从某种意义上讲，代码架构和组织架构管理有些思路是互通的，但是组织架构涉及到人，所以可变因素更多，搞起来就更难一些。

任意领域架构存在的意义是方便管理，是为了使较大规模系统中各机能模块的动作协调有效，但是为了支撑架构必然会产生一些内耗，这就要求我们在设计架构时，在保证动作正确的基础上要尽量减少内耗以提高单元和系统整体的工作效率。

对应到软件工程领域，增加了保证代码质量和可读性的作用；类似组件化这种，将大项目拆分成多个小项目，小项目间完全解耦，则分配到单一单元内部的复杂性降低了，沟通成本降下来了，效率自然也就上去了；再通过模块在不同项目中的复用以及统一规范的路由系统衔接，将公司所有的项目形成了一个有机的整体，又更进一步提供了承载大中台的能力。

slate组件化方案之前已经在“iWeekly周末画报”、“彭博商业周刊中文版”、“彭博商业周刊繁体版”、“艺术新闻”等项目中落地，但是一些思路和细节仍然在完善中，而且这些项目都是纯OC开发，所以方案并没有考虑混编和跨平台等情况。
       
### 前置问题
- - -
1. 随着项目人员的增加，会产生越来越多的沟通成本，降低开发效率

2. 代码越来越多，不同模块的界限划分的不清晰，复用的不规范等，会使我们对代码质量的把控越来越难，都耦合在一起的话，有时候想修改一个东西，可能牵一发动全身。
        
3. 公司的项目比较多，不同项目中所需的功能有重叠。

针对以上问题1、2中的情况，我们把大团队分为若干个小团队，然后进行解耦和职责分离，事情就好办多了，组件化方案能够从代码层面支撑我们对代码和开发团队的拆分。

针对问题3，我们可以把组件化上升为一个公司级的策略，利用组件化思维来支撑一个大中台服务，使得新项目仅通过配置文件的修改就可以迅速集成其他组件的相关功能，从而把精力聚焦在自己的业务组件上。同时由于制定了相关的路由通信规范，后期如果需要的话也能够很方便的实现不同项目的数据互通。

### 什么是组件化
- - -
组件化方案就是根据功能或业务的不同，将项目划分成一些不同的模块，核心基础是让各个模块都能在内部共享整个APP生命周期回调，和插件化或者工具SDK的概念不同，组件化中的每个模块都可以被看作一个APP，每个模块都能脱离整体，单独的进行维护和运行，在编译时期完全解耦；同时抽象出一个URI路由协议系统作为组件间通信的桥梁，使得项目中的推送、内嵌网页、通用链接、二维码扫描以及app内部的URI指令等能够得到解析和分发执行，从而可以像搭积木一样根据项目需要将各个模块以一种松散的形式组合成一个整体。

### 组件化模块的划分
- - -

![slate组件化项目结构图](https://github.com/SimonCY/slate-iOS/blob/master/imgs/slate_modules.png?raw=true)

模块一共分为四种，功能型Module、业务型module、XXMainModule 和 basicModule。

每个项目必须包含一个XXMainModule和多个其他功能型模块或业务型模块，XXMainModule用于在程序启动后的didFinishedLaunching中提供一个RootViewController，本质也是一个业务型模块，但是我们规定APP启动后由他来提供项目的视图入口。

我们注意区分工具SDK和功能型模块的区别：

1. 工具sdk作为depency，代码可以在各个模块中直接引用。

2. 功能型模块我们可以把他看成一个APP，其中的代码不能在其他模块中直接引用，但是能承载一部分功能，保证模块集成后，外部无需引用或修改代码就使得这部分功能有效。例如推送系统，内部包含推送接收，解析，uri分发等功能，集成后外部无需修改任何代码便使得项目接入了推送功能。

业务型模块本质上也是功能模块，只不过用于承载项目中的具体业务，一般情况下无法被其他项目复用（但也有特殊情况比如美团APP中的外卖业务和美团外卖APP，今日头条中的视频和西瓜视频，淘宝中的天猫和天猫APP等）。

所有的模块都依赖于basic模块，basic模块中包含组建化的基础设施，APP生命周期的分发、路由系统、内置浏览器、基础工具类等等。

如果涉及多个项目使用同一模块，这个模块必须对任何项目都是无差别对待的，如果某个模块需要针对不同项目进行不同配置（例如社会化模块中的第三方平台的Appkey或AppId等），须在slatefile中进行配置，然后由模块启动后，内部自己去读取slatefile。

### 模块内部结构
- - -
前面我们提到，模块和插件的概念不太一样，每个模块都可以看做是一个独立的APP，能单独进行维护和运行，除了所有模块都依赖于Basic模块以外，大家不分主次。

每个模块有两个代码入口类，一个Module类，用于接收APP生命周期事件，可以看成是默认项目结构下的APPDelegate；一个URIHandler类，用于接收路由系统分发来的URI指令。每个模块包含一个Resouces目录用于存放资源文件bundle，每个模块有自己单独的asset进行图片资源的管理。其他为正常结构下的代码。

每个模块维护一个URI接口文档，列出模块内支持的URI接口地址、参数表、响应形式或响应数据格式

### slatefile配置文件
- - -
slatefile是一个项目目录下的配置文件，所有模块可访问。

slatefile中记录内容包括：

1. 需要集成的模块列表，git地址、对应分支信息。

2. 此项目在运行时的灵活配置项，例：需要访问的域名、QQ或者微信的APPkey等。

slatefile用于在APP自动化模块集成、程序运行阶段提供项目级的配置项数据支持。
### 模块的集成
- - -

![slate项目目录示例截图](https://github.com/SimonCY/slate-iOS/blob/master/imgs/slate_project_shot.png?raw=true)

模块通过cocoapod本地联调指定分支形式进行集成（无需再提供一个索引仓库，且对模块内代码修改可以直接进行git提交，无需发布pod版本）。

由脚本读取slatefile中的信息，了解需要集成的模块，自动拼接对应的git仓库地址，进行clone，自动配置podfile，自动进行pod install集成。

Pods目录和各模块目录不加入当前Project的版本控制。

### 组件间的通信—APP生命周期事件的分发
- - -
![APP生命周期事件处理流程图](https://github.com/SimonCY/slate-iOS/blob/master/imgs/slate_app_event.png?raw=true)

项目中通过在basic模块中重写APPDelegate，进行：

1. 在APP启动时，根据slatefile配置文件实例化并持有已集成的各模块的moudle类。

2. 接收系统框架发出的APP生命周期事件，然后分发至已持有的各模块Module对象。

响应优先级根据slatefile中的模块列表数组顺序进行排序。

业内另一种解决方案为，无需设置slatefile这种配置文件，在各Module类的load方法中由其自身进行饿汉式创建，并向重写的APPDelegate进行注册，等待APP声明周期事件的分发，此种方法在效率上和我们的方法几乎没有差异，但是不方便在项目层面去统一设定响应优先级。

### 组件间的通信—路由系统的设计
- - -

![路由系统工作流程图](https://github.com/SimonCY/slate-iOS/blob/master/imgs/slate_router.png?raw=true)

##### URI的格式遵循URL协议:
- scheme://host/path?param_key0=param_value0&m_key1=param_value1
例： slate://com.mm.community/userInfo?sylte=1&color=green

代表访问了community下的 UserInfo URI资源

为了兼容之前的版本，这里我加了对 slate://userInfo?sylte=1&color=green 格式的支持，调用此格式Router内部会自动寻址找到所有响应这个URI的模块，并选择其中响应优先级最高的模块作为响应者。

##### 路由系统的工作主要包含两部分：
1. 接收并解析URI指令。

2. 将解析后的指令分发至各个模块的URIHandler。

解析阶段，因为URL协议只支持字符类型的参数，所以我们需要给APP内URI调用场景留出一个允许传递对象类型参数的接口，方便我们在不同模块间传递UIImage等对象类型的参数。对主URI中特殊符号不进行编码，对参数中的特殊符号进行一次编码，以满足类似嵌套一个带参数的URI作为参数这种需求。

分发阶段，OC项目下我们可以通过Runtime机制，动态的进行响应的URIHandler的懒加载创建、对响应方法的动态调用，但是纯swift项目由于无法进行动态创建和调用，所以需要我们有一个注册机制，在启动时各个模块为其支持的URI注册一个闭包，方便后续的URI分发调用，类似于在Router中维护了一个注册表。

OC项目下使用runtime本质也维护了注册机制，我们可以理解为各个模块Module类objc结构中的MethodList就是分布式的注册表，Module内部声明了这个方法就代表其对应的URI已经注册，这样更高效一些。

APP生命周期事件的分发是无具体的目标路径的，路由URI的分发可以有具体的目标路径，所以APP生命周期事件分发者必须要有一个统一的表来明确分发至何处（无论是根据slatefile创建，还是各个Module在load中自行创建，最后都需要由重写的APPDelegate对其进行引用，形成一个List），而Router可以根据URI精准定位到需要响应的URIHandler的指定方法，配合runtime，注册表可以只用于判断是否可以响应。

路由事件目前只用于：

- 页面跳转

- 模块间数据交互

不可用于返回一个本模块的自定义对象在另一个模块中持有使用，违背了解耦的基本原则。

### 使用中小tip（复杂APP视图上下文情况下的页面跳转）
- - -

由于由URI路由协议联系起来的各个模块都呈弱相关，且结合的比较随意，所以当一条URI过来时，我们可能需要跳转到一个新的页面，但是当前页面的情况可能就相对复杂了，可能是基于NavigationController管理的，可能是一个模态视图，等等，而且页面跳转时的电池栏、导航栏的显示和隐藏的衔接也是问题，所以推出了https://github.com/SimonCY/CYCustomedNavigation，主要包含：

1. 使每个ViewController独立管理自己的导航栏。
2. 同时通过自定义Present动画来模拟源生push，满足用户交互习惯，同时很方便进行新跳转动画的拓展。
3. 提供自主便利寻找当前顶部控制的API，提供dimiss至任意视图的API。

具体使用时，项目中所有的页面都通过present的形式弹出，同时兼容了针对 UIModalPresentationStyle 的不用设置，满足系统对视图树的自动优化策略。

### 组件化的弊端
- - -

- 规范问题，组件接口命名、回调的状态码以及返回信息格式需要严格设定并遵循规范。

- 组件协同开发阶段可能比较痛苦，组件间的协作在某种程度上类似于前后台联调，如果一个需求较为长尾，涉及的组件很多那很大概率产生的沟通成本会大于新造轮子成本，而且无脑的大量频繁的封装调用，层级关系和代码效率也会近乎失控，所以组件设计时一定要结合实际业务和团队现状考虑周全，组件并不是分的越细越多越好。

- 具体到产品同学提需求的时候，需要有了解架构的技术同学参与、协助将对应的开发任务分配到相应模块的开发组，这在一定程度上也增加部分沟通成本。
        
### 组件化的优势
- - -

- 开头提到的可以将大团队隔离成多个小团队，在内部提升开发效率以及利于对代码质量的把控，是对不同模块的代码、开发人员甚至业务人员的解耦。

- 路由系统跳转页面对服务器友好，方便由服务器灵活控制页面的跳转

- 支撑大中台战略，各个功能模块能够在各个项目中快速集成和响应业务

- 适合新的业务形式初期放在主站中试错和导流  验证可行和积累种子用户后迅速的独立出APP

- 能够使业务同学专注业务，使架构同学专注架构，使得某一领域同学专注在他的模块内部进行一些深度的技术研究，未来可能向集团其他部门做技术输出，实现事业群间的数据互通（当然这个推行的阻力主要不在于技术）

### 总结
- - -

综合来说，在管理得当的前提下组件化是利大于弊的，但是对于小型团队、小项目来说，个人认为并没有必要一定在内部支撑一个组件化方案，不应该把简单的事情复杂化。之前的团队规模比较小，项目也比较小，但是项目的数量较多且业务大多相似或者相同，但是我们可以综合起来把这些小项目看成一个较大的整体，这时组件化就派上用场了。所以slate方案的初衷是为了在投入较少的人力的前提下，来支撑“iWeekly”、“彭博商业周刊”以及“艺术新闻”等这些业务相似或者相同的APP的迭代开发，即凸显的是组件化中各个功能模块在不同项目中的复用的能力而非解耦能力。

当然，不论何时，框架选型工作都应从当前的团队规模、业务形态等具体情况出发，因地制宜。
