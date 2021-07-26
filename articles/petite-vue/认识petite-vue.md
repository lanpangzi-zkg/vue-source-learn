petite-vue还算是比较新的一个框架，尤雨溪2021年6月30号才初始化项目，经过几天密集的代码提交后，有二十多天已经没有更新了，看得出已经比较稳定了，本文不打算详细介绍petite-vue是干嘛的，有啥优势，关于这些可以查看github官方介绍(https://github.com/vuejs/petite-vue)，首先来看看怎么跑一个hello world吧。
```
<div v-scope>{{msg}}</div>
<script src="https://unpkg.com/petite-vue"></script>
<script>
    PetiteVue.createApp({ msg }).mount()
</script>
```
如果你熟悉vue，那么对petite-vue的用法就很熟悉了，毕竟师出同门，当然还有一些个性化的语法，如上面的v-scope；对petite-vue有了简单的认识后，我们就模仿上面的示例，来实现一个看起来一样的代码吧，其中我们要实现如下几个关键部分：

## PetiteVue
PetiteVue是一个全局对象，包含createApp这个重要的API，因此可以像下面这样声明:
```
const PetiteVue = {
    createApp(scope) {
        ...
    }
};
```
## createApp
createApp是一个函数，入参可以接收一个表示组件数据值的对象，同时需要返回一个包含mount函数的对象，我们在上一步的基础上接着丰富createApp函数吧：
```
const PetiteVue = {
    createApp(scope) {
        const appContent = {
            scope: scope,
        };
        const app = {
            context: appContent,
            mount() {
                ...
            }
        };
        return app;
    }
};
```
## mount
mount根据字面意思，就是挂载我们的组件了，这里我们只是简单的将msg渲染到页面上，要实现这一目标，我们要遍历div的DOM结构，找到{{插值}}的地方，然后用scope的值去填充文本，说完了思路，接下来就实现吧，这里我们新增两个遍历DOM的函数walk和walkChildren：
```
function walk(node, context) {
    const { nodeType } = node;
    if (nodeType === 1) { // Element
        return walkChildren(node, context);
    }
    if (nodeType === 3) { // Text
        ...
    }
}
function walkChildren(node) {
    let child = node.firstChild;
    while(!child) {
        walk(child);
        child = child.nextSibling;
    }
}
const PetiteVue = {
    createApp(scope) {
        const appContent = {
            scope: scope,
        };
        const app = {
            context: appContent,
            mount() {
                const root = document.querySelector('[v-scope]');
                if (!root) {
                    console.warn('请提供有v-scope属性的html标签');
                    return;
                }
                walk(root, appContent);
                root.removeAttribute('v-scope');
            }
        };
        return app;
    }
};
```
通过walk和walkChildren递归，可以遍历所有DOM节点，这里我们只关心Text节点，上面的代码还没实现具体逻辑，先不急，把架子搭起来，后面再实现。
## v-scope
v-scope是标记根组件的自定义属性，petite-vue支持多个根组件节点，在本篇实现中就先实现一个吧，尽量保持简单些；通过document.querySelector获取到根节点引用，它就作为遍历DOM的起点，当然最后要把v-scope属性删除，上面的代码已经实现了，这里多废话几句。
## {{}}
{{}}是我们自定义的插值语法，因此需要在walk遍历过程中去识别和解析出来，识别还是很简单的，就判断文本是不是{{xx}}格式的，通过一个简单的正则/{{([^]+?)}}/就可以判断，这里简单说一下正则表达式吧，[^]+?表示匹配任意字符，但是尽量少匹配，外面的括号是一个分组，会提取出{{}}里面的表达式，最后前后需要有{{}}包裹住，还是比较好理解的，现在动手实现具体的逻辑吧：
```
const RE = /{{([^]+?)}}/;
function walk(node, context) {
    const { nodeType } = node;
    if (nodeType === 1) { // Element
        return walkChildren(node, context);
    }
    if (nodeType === 3) { // Text
        const text = node.textContent;
        const match = text.match(RE);
        if (match) {
            const exp = match[1].trim(); // 删除表达式前后的空白字符
            node.textContent = context.scope[exp];
        }
    }
}
function walkChildren(node) {
    let child = node.firstChild;
    while(!child) {
        walk(child);
        child = child.nextSibling;
    }
}
const PetiteVue = {
    createApp(scope) {
        const appContent = {
            scope: scope,
        };
        const app = {
            context: appContent,
            mount() {
                const root = document.querySelector('[v-scope]');
                if (!root) {
                    console.warn('请提供有v-scope属性的html标签');
                    return;
                }
                walk(root, appContent);
                root.removeAttribute('v-scope');
            }
        };
        return app;
    }
};
```
现在可以在浏览器里面跑起来了，看下效果吧，嗯，跟petite-vue的例子看起来差不多了，到这里我们就基本达成了最初的目标了，实现了一版很简陋的看起来差不多的框架。

# 继续完善
从实现来看当匹配到插值语法的时候，我们直接把文本节点的内容全部替换了，如果我们的文本是这样的格式呢："this is content: {{msg}} is't over"，那么最终渲染的还是只有msg的状态值，其他都丢失了，这样显得有点糟糕，我们就乘胜追击，再完善一下吧。首先分析一下为了实现文本完整的渲染，我们要将静态的文本和插值文本提取出来，然后再拼接起来才是最终符合预期的结果，从左到右依次解析文本，"this is content: {{msg}} is't over"需要分成三部分，分别是["this is content: ", "{{msg}}", " is't over"]，msg经过转换后变成["this is content: ", "{hello world!", "is't over"]，最后拼接起来回填到文本节点就可以了：
```
 function walk(node, context) {
    const { nodeType } = node;
    if (nodeType === 1) { // Element
        return walkChildren(node, context);
    }
    if (nodeType === 3) { // Text
        const text = node.textContent;
        let i = 0; // 保存上一个匹配{{}}格式的字符结束索引
        if (text.includes('{{')) { // 先判断是否有"{{"字符，有才进行下面的判断
            let match = null;
            const segments = []; // 保存所有截断的文本
            while ((match = RE.exec(text))) {
                segments.push(text.slice(i, match.index)); // {{之前的字符
                i = match.index + match[0].length;
                const exp = match[1].trim(); // 删除表达式前后的空白字符
                segments.push(context.scope[exp]); // msg的值求得之后，放入数组中便于后面拼接
            }
            segments.push(text.slice(i)); // 最后一个}}后面的字符
            node.textContent = segments.join('');
        }
    }
}
```

# 支持表达式
通过拼接字符串的方式我们完成了渲染的基本要求，但是熟悉vue语法的同学会说，双花括号内部是支持js表达式的，既然实现到这里了，我们就支持一下表达式吧，首先分析一下，表达式里面的标识符指向scope对象的属性值，一个还好说，那么两个怎么通过简单的方式去实现呢，挨个挨个去把标识符提取出来，然后计算再合并么，想想都麻烦，那有没有简单的方式呢，我都这么说了，当然是有的，先看下实现原理吧:
```
function createFunc(exp) {
    return new Function(`scope`, `with(scope) { return (${exp}) }`)
}
const f = foo('a + b');
f({ a: 1, b: 2 });
```
通过createFunc创建一个新的函数，with将exp表达式的作用域限定在scope中，这样当执行a+b的时候，相当于scope.a + scope.b，最后将结果返回，最终执行的函数如下所示：
```
(function(scope) {
    with(scope) {
        return (a + b);
    }
})({a: 1, b: 2})
```
知晓了原理之后，我们就补齐表达式的计算吧：
```
function createFunc(exp) {
    return new Function(`scope`, `with(scope) { return (${exp}) }`);
}
...
function walk(node, context) {
    const { nodeType } = node;
    if (nodeType === 1) { // Element
        return walkChildren(node, context);
    }
    if (nodeType === 3) { // Text
        const text = node.textContent;
        let i = 0;
        if (text.includes('{{')) {
            let match = null;
            const segments = []; // 保存所有截断的文本
            while ((match = RE.exec(text))) {
                segments.push(text.slice(i, match.index));
                i = match.index + match[0].length;
                const exp = match[1].trim(); // 删除表达式前后的空白字符
                segments.push(createFunc(exp)(context.scope)); // createFunc(exp)生成函数，再将scope传入执行
            }
            segments.push(text.slice(i));
            node.textContent = segments.join('');
        }
    }
}
...
```
现在我们写的第一版框架就完成啦，完整的v1版本代码可点击<a href="../../code/petite-vue/v1.html">这里</a>，当然现在功能十分有限，没有其他指令集，没有响应式，不过作为学习petite-vue的第一步，已经迈出去啦，给自己一个赞吧，持之以恒，终会有收获的。这里预告一下第二篇的内容，我们将分析和实现响应式方面的内容。

