title: ElasticSearch kibana Dashboard 定义与使用
date: 2015-12-03 15:31:45
tags: [ElasticSearch, kibana]
---
以前在骡迹物流的时候只是玩儿了一下kibana简单的功能，并没有投入使用，当时也基本上玩儿遍了各种可视化，就是么有弄dashboard。
在美团这边前两天被拉进elasticsearch的群里，然后收到一个kibana的地址，今儿下午尝试性地玩儿了一下，还蛮爽的，有些惊异。

dashboard是一个画布，然后这个画布上需要放置很多的visualize，也就是说在弄dashboard之前是需要先准备好visualize的，它由visualize和search组成。

### 1. 准备源材料

创建并保存visualization和search

![](/imgs/es/准备源材料.png)

### 2. 新建dashboard

![](/imgs/es/新建dashboard.png)

### 3. 为dashboard添加元素

![](/imgs/es/创建dashboard.png)

### 4. 调整后保存

![](/imgs/es/保存.png)

好了，这已经非常好看了，完全可以直接看了。以后再有运营提数据需求，我就直接创建search或者visualization后，生成dashboard。可是问题又来了，咋给他看呢？总不能让所有的运营人员都登陆kibana页面来瞅吧，那样的话还得加权限神马的，比如对个别页面有只读权限，其他页面或者操作没有任何权限才对。静静地思考，kibana和elasticsearch已然有了思路，是不可能倒在权限这个小问题上的，肯定有一种合理的方式去分享给需求端看。fun，找找看~

### 5. dashboard组件化
叫我脑残，功能就放在眼前
![](/imgs/es/share.png)

获取到iframe后，自然就会想到，这个东西是可以嵌入到我们自己的网页中的。如此一来的话，就可以直接将数据存储在elasticsearch中，之后在elasticsearch和kibana之上建立自己的search，visualization，dashboard。然后将最终的dashboard嵌入到应用页面。

还是有些问题，直接在浏览器访问这个dashboard的url的话会发现其实就是kibana那个页面，但是放进iframe之后就成了不可编辑的状态，而且只有组件，上面不会有kibana的head，这个状态比较理想。可是右击查看源代码的话，还是可以看到kibana的页面接口URL，如果直接访问还是可以直接进行编辑。

查了一下权限相关

- https://github.com/floragunncom/search-guard
- http://drops.wooyun.org/tips/8129
- http://3gods.com/2015/09/10/Shield%20Control.html
