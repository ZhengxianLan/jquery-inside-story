## 《jQuery技术内幕读书笔记》

虽然随着 Angular、React、Vue 等框架的出现，jQuery 已经慢慢退出了历史舞台，但这并不说明 jQuery 本身不优秀，它是前端发展过程中十分重要的一环，重要性可能高过 Angular。jQuery 本身的架构设计、兼容性设计等都是很棒的，很有学习的价值。因此决定花一点时间把这本 600 页的书看一遍，版本比较旧，1.10.1，不过对于学习源码来说影响不是很大，因为核心是一样的，另外旧版本的好处是兼容的范围比较广，也就能够学到更多兼容性方面的知识。

> 书上用的是 1.7.1，本人看源码的是 1.10.1。因此部分内容和我自己添加的源码分析可能会有小出入。

### 目录

1. [总体架构](./lib/总体架构.md)
2. [构造 jQuery 对象](./lib/构造 jQuery 对象.md)
3. 底层支持模块（底层支持模块写得详细程度会比较低）
   1. [选择器 Sizzle](./lib/选择器 Sizzle.md)
   2. 异步队列 Deferred Object
   3. [数据缓存 Data](./lib/数据缓存 Data.md)
   4. 队列 Queue
   5. [浏览器功能测试](./lib/浏览器功能测试.md)
   6. [番外：标准模式和诡异模式](./lib/标准模式和诡异模式.md)
4. 功能模块
   1. [属性操作](./lib/属性操作.md)
   2. [事件系统](./lib/事件系统 Event.md)
   3. DOM 遍历
   4. DOM 操作
   5. 样式操作
   6. 异步请求 Ajax
   7. 动画 Effects
