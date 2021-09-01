# v-for
紧接着前面的系列文章，今天的目标就是学习v-for这个指令，petite-vue中的用法比较灵活，首先就来认识一下语法吧。
## 认识语法
1. ...of...
```
<ul>
  <li v-for="item of list" :key="item.id">
    <input v-model="item.text" />
  </li>
</ul>
```
2. ...in...
```
<ul>
  <li v-for="item in list" :key="item.id">
    <input v-model="item.text" />
  </li>
</ul>
```
3. ({ ... }, index) in ...
```
<ul>
  <li v-for="({ id, text }, index) in list" :key="id">
    <div>{{ index }} {{ { id, text } }}</div>
  </li>
</ul>

```
第一种和第二种差不太多，第三种使用了解构赋值的方式；此外，v-for的目标数据支持数组、对象和数字三种格式，相比前面我们分析的指令，v-for显得又不太一样，那么如果要我们来实现，该怎么设计呢，先只考虑第一种使用方式(...of...)，我希望的样子应该是这样的：
```
const scope = { list: [...] };

function for_dir(ele) {
  const valueExp = 'item';
  const sourceExp = 'list';
  const keyExp = 'item.id';
  for (let i = 0; i < scope[sourceExp].length; i++) {
    createForItem(ele, { [valueExp]: scope[sourceExp][i] }, i);
  }
}
function createForItem(ele, source, index) {
  // ...
}
```
首先我需要解析出v-for的指令值，分解出valueExp、sourceExp这些关键信息，然后通过循环创建每一个子项，这里通过包装一个与每个子项对应的数据对象，构建一个相对隔离的上下文，当然在petite-vue里面是通过childContext来实现的。除此之外，还有key这个很重要的属性，不管是在vue还是react中，针对列表渲染都是一个性能优化的重要手段，那么在更新的时候通过比对key来减少DOM操作，后面将介绍petite-vue是怎么实现性能优化的。

## 指令语法解析
v-for指令值其实可以分为两个部分，in/of作为分隔符，前面一部分是子节点的上下文需要的数据映射关系，后一部分是主数据源映射关系，那么首先通过正则分离出这两部分吧；
```
const forAliasRE = /([\s\S]*?)\s+(?:in|of)\s+([\s\S]*)/;
```
两个分组对应我们关注的两部分前后两部分指令值，这里稍微说一下这条正则表达式吧，`in|of`匹配v-for的固定语法，`(?:in|of)`表示不分组只匹配特定结构，因此最终匹配的结果只有`([\s\S]*?)`和`([\s\S]*)`这两个分组，`\s\S`会匹配任意字符，`*?`代表尽量少匹配，因为in/of前后会有空格，这才完成了第一步，接下来要针对valueExp(第一个分组匹配的内容)进行更加精细的判断，这里再回顾一下valueExp可能的几种格式：

1. { id, text }【, index, objIndex】
2. [ id, text ]【, index, objIndex】
3. item【, index, objIndex】

针对前面两种格式，需要以`}]`作为分隔线，前面一部分是结构化赋值，后面是索引，结构化赋值的对象可能是数组，这里要判断一下，然后将里面赋值的标识符提取出来放入数组中；后面的索引包含index、objIndex，作为选填项；
```
input: 
`({ id, text }, index) of list`
output: 
let sourceExp = 'list';
let valueExp = '{ id, text }';
let isArrayDestructure = false;
let destructureBindings = ['id', 'text'];
```
至此就完成了指令语法的解析工作，代码就不贴了，主要是那几个比较复杂的正则表达式，大家可以查看[这里](https://github.com/lanpangzi-zkg/vue-source-learn/blob/main/code/petite-vue/v4.html)。
## 循环创建
前面也提到过，针对每一个子项需要有单独的上下文(childContext)，而childContext保存着数据对象，而childContext和当前的context又是什么关系呢，其实就是childContext.scope__proto__指向context.scope，确保状态访问的有效性和有序性，接下来梳理一下大体流程：

1. 根据sourceExp获取数据源，判断数据源的类型；
```
 if (Array.isArray(source)) {
    for (let i = 0; i < source.length; i++) {
        ...
    }
} else if (typeof source === 'number') {
    for (let i = 0; i < source; i++) {
        ...
    }
} else if (isObject(source)) {
    for (let key in source) {
        ...
    }
}
```
2. 创建childContext
```
const parentScope = ctx.scope;
const mergedScope = Object.create(parentScope); // 建立mergedScope与parentContext.scope的原型联系
Object.defineProperties(mergedScope, Object.getOwnPropertyDescriptors(data)); // 用上一步获取到的子项数据data填充mergedScope对象
const reactiveProxy = reactive(new Proxy(mergedScope, { // mergedScope包装成响应式
    set(target, key, val, receiver) {
        // when setting a property that doesn't exist on current scope,
        // do not create it on the current scope and fallback to parent scope.
        if (receiver === reactiveProxy && !target.hasOwnProperty(key)) {
            return Reflect.set(parentScope, key, val);
        }
        return Reflect.set(target, key, val, receiver);
    }
}));
return {
    ...ctx,
    scope: reactiveProxy,
}  
```
3. 建立index和key之间的映射关系
index在第一步循环过程中可以获得，key如果没有设置，默认就是index，index和key都准备好后，通过Map保存，方便后面更新优化；

4. 创建DOM节点
源代码中引入Block来进行动态节点管理，负责保存节点模板及父节点，还有插入、删除等操作，比较简单，就不多说，具体可以查看源码；

## 更新优化
更新的时候，要点就是比较前后两次渲染的节点差异，主要分为新增、删除和更新，首先假定更新前后两次数据如下：
```
---mount---
blocks = [b1, b2, b3];
keyToIndexMap = { b1->0, b2->1, b3->2 }; // key<->index映射关系

---update---
blocks = [b1, b2, b3]; // mount时block数组
prevKeyToIndexMap = { b1->0, b2->1, b3->2 }; // mount时的映射关系
nextBlocks = []; // 当前更新需要渲染的block，后面会通过算法填充，最终结果[b1, b3, b4]
keyToIndexMap = { b1->0, b3->1, b4->2 }; // 本次update根据状态对象新生成的映射关系对象
```
首先我们考虑一下删除的情况，通过keyToIndexMap和blocks即可判断，如果blocks每一项的key没有包含在keyToIndexMap中，那么意味着mount时的block需要删除，就像例子中的b2，代码如下：
```
for (let i = 0; i < blocks.length; i++) {
  if (!keyToIndexMap.has(blocks[i].key)) {
    blocks[i].remove();
  }
}
```
我们再来考虑下新增，要判断是否新增还是比较简单的，key没有在prevKeyToIndexMap就对了，然后创建新的Block对象放入nextBlocks中保存起来，这里有个问题就是对于新增的这个节点，插入的具体位置在哪呢，有可能插入末尾，有可能插入中间，那么需要一个定位的基准点，这个基准点肯定需要前后两次对比才能确定下来，也必然和新插入的节点相邻吧，就像insertBefore那样，具体的算法后面再讲，到此就剩下最后一种情况了--更新。更新可能时位置变动，可能是ui变动，位置变动通过比对这个block的key在keyToIndexMap和prevKeyToIndexMap的索引就可以得出，ui变动就是状态值的变动，将scope合并即可。分析到这里，流程应该比较清楚了，还有个问题就是前面说的基准点，接下来就在代码里寻找答案吧；
```
const nextBlocks = [];
let i = childCtxs.length; // childCtxs每次更新都会根据状态值重新生成，保存本次需要渲染的上下文信息
while (i--) {
  const childCtx = childCtxs[i]; // 当前判断的上下文
  const oldIndex = prevKeyToIndexMap.get(childCtx.key); // 上下文的key和对应的block对象的key相同
  const next = childCtxs[i + 1]; //
  const nextBlockOldIndex = next && prevKeyToIndexMap.get(next.key); // 下一个上下文在上一次渲染的索引
  const nextBlock =
    nextBlockOldIndex == null ? undefined : blocks[nextBlockOldIndex]; // 如果nextBlockOldIndex存在，说明childCtx对应的block在上一次渲染时，后面有block，以此为基准点
  if (oldIndex == null) { // key在prevKeyToIndexMap不存在，那么必然是新增
    // new
    nextBlocks[i] = mountBlock(
      childCtx,
      nextBlock ? nextBlock.el : anchor
    );
  } else {
    // update
    const block = (nextBlocks[i] = blocks[oldIndex]); // 更新，直接复用存在的block
    Object.assign(block.ctx.scope, childCtx.scope); // scope合并，确保ui更新
    if (oldIndex !== i) { // 位置发生了变化
      // moved
      if (blocks[oldIndex + 1] !== nextBlock) {
        block.insert(parent, nextBlock ? nextBlock.el : anchor);
      }
    }
  }
}
blocks = nextBlocks;
```
通过上面的代码分析，nextBlock是很重要的定位点，确定新插入和更新的位置，至此就分析完毕v-for指令的实现了，具体实现的完整代码点击[这里](https://github.com/lanpangzi-zkg/vue-source-learn/blob/main/code/petite-vue/v4.html)。

最后再针对上面的更新算法说明一下，源码中其实是有点问题的，列表渲染的顺序是不稳定的，例如mount时数据源是[1,2,3]，渲染顺序符合预期(1,2,3)，但当update时，如果数据源变成了[4,5,6]，界面渲染的最终效果却是倒序的(6,5,4)，为什么会这样，因为当没有nextBlock时，插入的基准节点都是anchor，插入流程变成：`6 anchor`，`6 5 anchor`，`6 5 4 anchor`，因此最终我们看到的就是654了，那怎么改呢，anchor每一次循环后都需要指向childCtx的渲染节点，这样就能保证顺序了。