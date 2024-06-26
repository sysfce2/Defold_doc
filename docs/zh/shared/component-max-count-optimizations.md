## 最大组件数优化限制
配置文件 *game.project* 包含针对各种资源同时存在的最大数目的配置项, 容器大都是每个载入的集合 (也叫做游戏世界). Defold 引擎使用这些最大值来预分配内存以避免游戏运行时内存动态分配及内存碎片化.

用于表示组件和其他资源的 Defold 数据结构优化为尽可能少用内存, 但当配置这些最大值时仍要留意是否保证了满足游戏运行的需求.

另一方面 Defold 编译处理会分析游戏内容, 如果明确分析出必要的内存需求则会覆盖最大值设置:

* 如果集合不含工厂组件则仅分配各个组件必须的内存忽略最大值设置.
* 如果集合包含工厂组件则分析工厂可实例化的游戏对象组件的必要内存并为其实例化分配足够内存.
* 如果集合里包含了工厂或者集合工厂, 并且这些工厂开启了 "Dynamic Prototype" 选项, 则该集合自动使用最大组件数.