# 目标
条件渲染使用频率还是蛮高的，今天就来实现一版自己的v-if。有了前面两篇文章做铺垫，对于v-if这个新指令，无非就是通过不同的指令函数处理，当然内部的处理逻辑是不同的，整理流程在第二篇文章的基础上是不变的，接下来就开始吧。

# v-if
v-if指令有几种组合方式：`v-if`、`v-if/v-else`、`v-if/v-else-if`、`v-if/v-else-if/v-else`，这就跟js的if/else类似的，首先来看一下一个简单的例子：
```
<div id="app">
  <div v-if="open">open</div>
  <div v-else>close</div>
</div>
```
看到这些指令，是不是很熟悉，跟js里面的if/else很像嘛，比如这样：
```
<script>
  let open = true;
  const parentEle = document.querySelector('#app');
  parentEle.innerHTML = '';
  if (open) {
    const newEle = document.createElement('div');
    newEle.textContent = 'open';
    parentEle.appendChild(newEle);
  } else {
    const newEle = document.createElement('div');
    newEle.textContent = 'else';
    parentEle.appendChild(newEle);
  }
</script>
```
那么框架就为我们做了以上这些事情，现在我们就来探究一下petite-vue是怎么实现的。

## 指令解析
v-if指令稍微有点特殊，因为是有前后关联的一套组合语法，那么在解析指令的时候，需要往后判断同级子元素，例如现在我们找到了第一个拥有v-if属性的元素A，后面的元素名称分别为B、C、D、E...，那么判断有如下几种情况：

1.) B有v-else，完毕；

2.) B有v-else-if，继续搜索C，重复1、2步骤；

3.) B没有v-else/v-else-if属性，完毕；

了解析规则之后，我们尝试自己实现这个过程吧，模板长下面这个样子：
```
<div id="app" v-scope="{ open: false, elseOpen: false }">
  <div v-if="open">open</div>
  <div v-else-if="elseOpen">elseOpen</div>
  <div v-else>close</div>
</div>
```
这里我们通过v-scope来初始化状态对象，然后通过三个分支来实现条件渲染，接下来就是脚本实现了；
```
<script type="text/javascript">
  const root = document.querySelector('#app');
  const scope = new Function(`return (${root.getAttribute('v-scope')});`)();
  let currentEle = root.firstElementChild;
  while(currentEle) {
    if (currentEle.hasAttribute('v-if')) { // 发现节点元素有v-if属性，就交由if指令函数处理
      if_dir(currentEle);
    }
    // 其他逻辑
    currentEle = currentEle.nextElementSibling;
  }
  function executeExp(exp) {
    return new Function('scope', 'exp', `with(scope) { return (${exp}); }`)(scope, exp);
  }
  function getAttr(ele, attrName) {
    let attrValue = '';
    if (ele.hasAttribute(attrName)) {
      attrValue = ele.getAttribute(attrName);
      ele.removeAttribute(attrName);
    }
    return attrValue;
  }
  function if_dir(ele) {
    const branchs = [{
      ele,
      exp: getAttr(ele, 'v-if'),
    }]; // 保存所有的分支节点
    let nextEle = ele;
    while((nextEle = nextEle.nextElementSibling)) {
      if (nextEle.hasAttribute('v-else')) {
        branchs.push({
          ele: nextEle,
        });
        break; // v-else表示分支结束，此时跳出循环
      } else if (nextEle.hasAttribute('v-else-if')) {
        branchs.push({
          ele: nextEle,
          exp: getAttr(nextEle, 'v-else-if'),
        });
      }
    }
    const parent = ele.parentElement;
    // 先删所有的分支节点
    branchs.forEach(({ ele }) => {
      parent.removeChild(ele);
    });

    // 根据表达式执行结果渲染目标节点
    for (let i = 0; i < branchs.length; i++) {
      const { ele, exp } = branchs[i];
      if (executeExp(exp) || !exp) { // v-else没有表达式，因此没有其他分支命中的话，就显示最后一个分支
        parent.appendChild(ele);
        break;
      }
    }
  }
</script>
```
了解了上面的版本后，就可以去看看petite-vue是怎么实现的了，当然我也提供了示例页面，点击[这里](https://github.com/lanpangzi-zkg/vue-source-learn/blob/main/code/petite-vue/v3.html)


