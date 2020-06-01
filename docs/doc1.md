---
id: doc1
title: Latin-ish
sidebar_label: Example Page
---

# 前言

最近po主写小程序过程中遇到一个拖拽排序需求. 上网一顿搜索未果, 遂自行实现.

这次就不上效果图了, 直接扫码感受吧.

![](https://img2018.cnblogs.com/blog/1058332/201909/1058332-20190921140750280-882186510.png)

# 灵感

首先由于并没有啥现成的小程序案例给我参考. 所以有点无从下手, 那就找个h5的拖拽实现参考参考. 于是在jquery插件网看了几个拖拽排序实现后基本确定了思路. 大概就是用 transform 做变换. 是的, 灵感这种东西就是借鉴过来的～～

# 确定需求

1. 要能拖拽, 毕竟是拖拽排序嘛, 拖拽肯定是第一位.
2. 要能排序, 先有拖拽后有天 ～～ 跑偏了, 拖拽完了肯定是要排序的要不然和movable-view有啥区别呢.
3. 能自定义列数以实现不同形式展现, 这里考虑到有列表排序, 相册排序等不同情况需要的列数不同的需求.
4. 没有bug, 呃呃呃, 这个我尽量.

# 实现思路

首先能拖拽的元素最起码都要是一样的大小, 至于不规则大小, 或者大小成倍数关系的均不在本次实现范围.

然后我们对应需求找解决方案:

## 拖拽实现

1. 使用 movable-view 实现拖拽, 这种方式简单快捷, 但是由于我们的灵感是使用 transform 做变换, 而这里 movable-view 本身也是用 transform 来实现的, 所以会有冲突, 遂弃之.

2. 使用自定义手势, 如 touchstart, touchmove, touchend. 对的又是这三个基佬, 虽然我们在做下拉刷新时候采用用了 movable-view 而抛弃这三兄弟. 但是是金子总会发光的, 今天就是你们三兄弟展示自身本领的时候了(真香警告). 废话有点多, 言归正传, 使用自定义手势可以方便我们控制每一个细节.

## 排序实现

排序是基于拖拽的, 通过上面 touchstart, touchmove, touchend 这三兄弟拿到触摸信息后动态计算出当前元素的排序位置,然后根据当前激活元素的排序位置去动态更换数组内其他元素的位置. 大概意思就是十个兄弟做一排, 老大起来跑到老三的位置, 老三看了看往前移了移, 老二看了看也往前移了移. 当然这是正序, 还有逆序, 比如老十跑到了老大的位置, 那么老大到老九都得顺序后移一个位置.

## 自定义列数

自定义列数, 到是没啥难度, 小程序组件暴露一个列属性, 然后把计算过程中的固定的列数改成该参数就可以了


# 实现分析

先上 touchstart, touchmove, touchend 三兄弟

## longPress 

这里为了体验把 touchstart 换成了 longpress 长按触发. 首先我们需要设置一个状态 touch 表示我们在拖拽了. 然后就是获取 pageX, pageY 注意这里获取 pageX, pageY 而不是 clientX, clientY 因为我们的 drag 组件有可能会有 margin 或者顶部仍有其他元素, 这时候如果获取 clientX, clientY 就会出现偏差了. 这里把当前 pageX, pageY 设置为初始触摸点 startX, startY. 

然后需要计算下初始化的激活元素的偏移位置 tranX 和 tranY, 这里为了优化体验在列数为1的时候初始化 tranX 不做位移, tranY 移动到当前激活元素中间位置, 多列的时候把 tranX 和 tranY 全部位移到当前激活元素中间位置.

最后设置当前激活元素的索引 cur 和 curZ(该参数用于控制激活元素z轴的显示时机, 具体参看 wxml 中代码以及 clearData 方法中对应的代码) 以及偏移量 tranX, tranY. 然后震动一下下 wx.vibrateShort() 体验美美哒.

```
/**
 * 长按触发移动排序
 */
longPress(e) {
    this.setData({
        touch: true
    });

    this.startX = e.changedTouches[0].pageX
    this.startY = e.changedTouches[0].pageY

    let index = e.currentTarget.dataset.index;

    if(this.data.columns === 1) { // 单列时候X轴初始不做位移
        this.tranX = 0;
    } else {  // 多列的时候计算X轴初始位移, 使 item 水平中心移动到点击处
        this.tranX = this.startX - this.item.width / 2 - this.itemWrap.left;
    }

    // 计算Y轴初始位移, 使 item 垂直中心移动到点击处
    this.tranY = this.startY - this.item.height / 2 - this.itemWrap.top;

    this.setData({
        cur: index,
        curZ: index,
        tranX: this.tranX,
        tranY: this.tranY,
    });

    wx.vibrateShort();
}
```

## touchMove

touchmove 每次都是故事的主角, 这次也不列外. 看这满满的代码量就知道了. 首先进来需要判断是否在拖拽中, 不是则需要返回.

然后判断是否超过一屏幕. 这是啥意思呢, 因为我们的拖拽元素可能会很多甚至超过整个屏幕, 需要滑动来处理. 但是我们这里使用了 catch:touchmove 事件所以会阻塞页面滑动. 于是我们需要在元素超过一个屏幕的时候进行处理, 这里分两种情况. 一种是我们拖拽元素到页面底部时候页面自动向下滚动一个元素高度的距离, 另一种是当拖拽元素到页面顶部时候页面自动向上滚动一个元素高度的距离.

接着我们设置已经重新计算好的 tranX 和 tranY, 并获取当前元素的排序关键字 key 作为初始 originKey, 然后通过当前的 tranX 和 tranY 使用 calculateMoving 方法计算出 endKey. 

最后我们调用 this.insert(originKey, endKey) 方法来对数组进行排序

```
touchMove(e) {
    if (!this.data.touch) return;
    let tranX = e.touches[0].pageX - this.startX + this.tranX,
        tranY = e.touches[0].pageY - this.startY + this.tranY;

    let overOnePage = this.data.overOnePage;

    // 判断是否超过一屏幕, 超过则需要判断当前位置动态滚动page的位置
    if(overOnePage) {
        if(e.touches[0].clientY > this.windowHeight - this.item.height) {
            wx.pageScrollTo({
                scrollTop: e.touches[0].pageY + this.item.height - this.windowHeight,
                duration: 300
            });
        } else if(e.touches[0].clientY < this.item.height) {
            wx.pageScrollTo({
                scrollTop: e.touches[0].pageY - this.item.height,
                duration: 300
            });
        }
    }

    this.setData({tranX: tranX, tranY: tranY});

    let originKey = e.currentTarget.dataset.key;

    let endKey = this.calculateMoving(tranX, tranY);

    // 防止拖拽过程中发生乱序问题
    if (originKey == endKey || this.originKey == originKey) return;

    this.originKey = originKey;

    this.insert(originKey, endKey);
}
```

### calculateMoving 方法

通过以上介绍我们已经基本完成了拖拽排序的主要功能, 但是还有两个关键函数没有解析. 其中一个就是 calculateMoving 方法, 该方法根据当前偏移量 tranX 和 tranY 来计算 目标key. 

具体计算规则:

1. 根据列表的长度以及列数计算出当前的拖拽元素行数 rows
2. 根据 tranX 和 当前元素的宽度 计算出 x 轴上的偏移数 i
3. 根据 tranY 和 当前元素的高度 计算出 y 轴上的偏移数 j
4. 判断 i 和 j 的最大值和最小值
5. 根据公式 endKey = i + columns * j 计算出 目标key
6. 判断 目标key 的最大值
7. 返回 目标key

```
/**
 * 根据当前的手指偏移量计算目标key
 */
calculateMoving(tranX, tranY) {
    let rows = Math.ceil(this.data.list.length / this.data.columns) - 1,
        i = Math.round(tranX / this.item.width),
        j = Math.round(tranY / this.item.height);

    i = i > (this.data.columns - 1) ? (this.data.columns - 1) : i;
    i = i < 0 ? 0 : i;

    j = j < 0 ? 0 : j;
    j = j > rows ? rows : j;

    let endKey = i + this.data.columns * j;

    endKey = endKey >= this.data.list.length ? this.data.list.length - 1 : endKey;

    return endKey
}
```

### insert 方法

拖拽排序中没有解析的另一个主要函数就是 insert方法. 该方法根据 originKey(起始key) 和 endKey(目标key) 来对数组进行重新排序.

具体排序规则:

1. 首先判断 origin 和 end 的大小进行不同的逻辑处理
2. 循环列表 list 进行逻辑处理
3. 如果是 origin 小于 end 则把 origin 到 end 之间(不包含 origin 包含 end) 所有元素的 key 减去 1, 并把 origin 的key值设置为 end
4. 如果是 origin 大于 end 则把 end 到 origin 之间(不包含 origin 包含 end) 所有元素的 key 加上 1, 并把 origin 的key值设置为 end
5. 调用 getPosition 方法进行渲染

```
/**
 * 根据起始key和目标key去重新计算每一项的新的key
 */
insert(origin, end) {
    let list;

    if (origin < end) {
        list = this.data.list.map((item) => {
            if (item.key > origin && item.key <= end) {
                item.key = item.key - 1;
            } else if (item.key == origin) {
                item.key = end;
            }
            return item
        });
        this.getPosition(list);

    } else if (origin > end) {
        list = this.data.list.map((item) => {
            if (item.key >= end && item.key < origin) {
                item.key = item.key + 1;
            } else if (item.key == origin) {
                item.key = end;
            }
            return item
        });
        this.getPosition(list);
    }
}
```

### getPosition 方法

以上 insert 方法中我们最后调用了 getPosition 方法, 该方法用于计算每一项元素的 tranX 和 tranY 并进行渲染, 该函数在初始化渲染时候也需要调用. 所以加了一个 vibrate 变量进行不同的处理判断.

该函数执行逻辑:

1. 首先对传入的 data 数据进行循环处理, 根据以下公式计算出每个元素的 tranX 和 tranY (this.item.width, this.item.height 分别是元素的宽和高, this.data.columns 是列数, item.key 是当前元素的排序key值)
    item.tranX = this.item.width * (item.key % this.data.columns);
    item.tranY = Math.floor(item.key / this.data.columns) * this.item.height;
2. 设置处理后的列表数据 list
3. 判断是否需要执行抖动以及触发事件逻辑, 该判断用于区分初始化调用和insert方法中调用, 初始化时候不需要后面逻辑
4. 首先设置 itemTransition 为 true 让 item 变换时候加有动画效果
5. 然后抖一下, wx.vibrateShort(), 嗯~, 这是个好东西
6. 最后copy一份 listData 然后出发 change 事件把排序后的数据抛出去

最后注意, 该函数并未改变 list 中真正的排序, 而是根据 key 来进行伪排序, 因为如果改变 list 中每一个项的顺序 dom结构会发生变化, 这样就达不到我们要的丝滑效果了. 但是最后 this.triggerEvent('change', {listData: listData}) 时候是真正排序后的数据, 并且是已经去掉了 key, tranX, tranY 的原始数据信息(这里每一项数据有key, tranX, tranY 是因为初始化时候做了处理, 所以使用时无需考虑)

```
/**
 * 根据排序后 list 数据进行位移计算
 */
getPosition(data, vibrate = true) {
    let list = data.map((item, index) => {
        item.tranX = this.item.width * (item.key % this.data.columns);
        item.tranY = Math.floor(item.key / this.data.columns) * this.item.height;
        return item
    });

    this.setData({
        list: list
    });

    if(!vibrate) return;

    this.setData({
        itemTransition: true
    })

    wx.vibrateShort();

    let listData= [];

    list.forEach((item) => {
        listData[item.key] = item.data
    });

    this.triggerEvent('change', {listData: listData});
}
```

## touchEnd

写了这么久, 三兄弟就剩最后一个了, 这个兄dei貌似不怎么努力嘛, 就两行代码？

是的, 就两行... 一行判断是否在拖拽, 另一行清除缓存数据

```
touchEnd() {
    if (!this.data.touch) return;

    this.clearData();
}
```

### clearData 方法

因为有重复使用, 所以选择把这些逻辑包装了一层. 

```
/**
 * 清除参数
 */
clearData() {
    this.originKey = -1;

    this.setData({
        touch: false,
        cur: -1,
        tranX: 0,
        tranY: 0
    });

    // 延迟清空
    setTimeout(() => {
        this.setData({
            curZ: -1,
        })
    }, 300)
}
```

## init 方法

介绍完三兄弟以及他们的表亲后, 故事就剩我们的 init 方法了.

init 方法执行逻辑:

1. 首先就是对传入的 listData 做处理加上 key, tranX, tranY 等信息
2. 然后设置处理后的 list 以及 itemTransition 为 false(这样初始化就不会看见动画了)
3. 获取 windowHeight 
4. 获取每一项 item 的宽高等属性 并设置为 this.item 留做后用
5. 初始化执行 this.getPosition(this.data.list, false)
6. 设置动态计算出来的父级元素高度 itemWrapHeight, 因为这里使用了绝对定位和transform所以父级元素无法获得高度, 故手动计算并赋值
7. 最后就是获取父级元素 item-wrap 的节点信息并计算是否超过一屏, 并设置 overOnePage 值

```
init() {
    // 遍历数据源增加扩展项, 以用作排序使用
    let list = this.data.listData.map((item, index) => {
        let data = {
            key: index,
            tranX: 0,
            tranY: 0,
            data: item
        }
        return data
    });

    this.setData({
        list: list,
        itemTransition: false
    });

    this.windowHeight = wx.getSystemInfoSync().windowHeight;

    // 获取每一项的宽高等属性
    this.createSelectorQuery().select(".item").boundingClientRect((res) => {

        let rows = Math.ceil(this.data.list.length / this.data.columns);

        this.item = res;

        this.getPosition(this.data.list, false);

        let itemWrapHeight = rows * res.height;

        this.setData({
            itemWrapHeight: itemWrapHeight
        });

        this.createSelectorQuery().select(".item-wrap").boundingClientRect((res) => {
            this.itemWrap = res;

            let overOnePage = itemWrapHeight + res.top > this.windowHeight;

            this.setData({
                overOnePage: overOnePage
            });

        }).exec();
    }).exec();
}
```

## wxml 

~~以下是整个组件的 wxml, 其中具体渲染部分使用了抽象节点 `<item item="{{item.data}}"></item>` 并传入了每一项的数据, 使用抽象节点是为了具体展示的效果和该组件本身代码解耦. 如果要到性能问题或者觉得麻烦, 可直接在该组件下编写样式代码.~~ 

最新实现中已经删除了抽象节点, 经测试抽象节点会在某些老款机型如: iphone 6s 及以下型号机器上产生巨大性能问题, 所以这里直接把渲染逻辑写入 wxml 中. 需要使用该组件直接修改 .info 部分样式和内容即可.

```
<view>
	<view style="overflow-x: {{overOnePage ? 'hidden' : 'initial'}}">
		<view class="item-wrap" style="height: {{ itemWrapHeight }}px;">
			<view class="item {{cur == index? 'cur':''}} {{curZ == index? 'zIndex':''}} {{itemTransition ? 'itemTransition':''}}"
				  wx:for="{{list}}"
				  wx:key="{{index}}"
				  id="item{{index}}"
				  data-key="{{item.key}}"
				  data-index="{{index}}"
				  style="transform: translate3d({{index === cur ? tranX : item.tranX}}px, {{index === cur ? tranY: item.tranY}}px, 0px);width: {{100 / columns}}%"
				  bind:longpress="longPress"
				  catch:touchmove="touchMove"
				  catch:touchend="touchEnd">
				<view class="info">
					<view>
						<image src="{{item.data.images}}"></image>
					</view>
				</view>
			</view>
		</view>
	</view>
	<view wx:if="{{overOnePage}}" class="indicator">
		<view>滑动此区域滚动页面</view>
	</view>
</view>
```

## wxss

这里我直接把 scss 代码拉出来了, 这样看的更清楚, 具体完整代码文末会给出地址

```
@import "../../assets/css/variables";

.item-wrap {
	position: relative;
	.item {
		position: absolute;
		width: 100%;
		z-index: 1;
		&.itemTransition {
			transition: transform 0.3s;
		}
		&.zIndex {
			z-index: 2;
		}
		&.cur {
			background: #c6c6c6;
			transition: initial;
		}
	}
}

.info {
	position: relative;
	padding-top: 100%;
	background: #ffffff;
	& > view {
		position: absolute;
		border: 1rpx solid $lineColor;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		overflow: hidden;
		padding: 10rpx;
		box-sizing: border-box;
		image {
			width: 100%;
			height: 100%;
		}
	}
}

.indicator {
	position: fixed;
	z-index: 99999;
	right: 0rpx;
	top: 50%;
	margin-top: -250rpx;
	padding: 20rpx;
	& > view {
		width: 36rpx;
		height: 500rpx;
		background: #ffffff;
		border-radius: 30rpx;
		box-shadow: 0 0 10rpx -4rpx rgba(0, 0, 0, 0.5);
		color: $mainColor;
		padding-top: 90rpx;
		box-sizing: border-box;
		font-size: 24rpx;
		text-align: center;
		opacity: 0.8;
	}
}
```

# 写在结尾

该拖拽组件来来回回花了我好几周时间, 算的上是该组件库中最有质量的一个组件了. 所以如果您看了觉得还不错欢迎star. 当然遇到问题在 issues 提给我就行了, 我回复还是蛮快的～～ 

~~还有就是该组件受限制于微信本身的 api 以及一些特性, 在超出一屏时候会无法滑动. 这里我做了个判断超出一屏时候加了个指示器辅助滑动, 使用时可对样式稍做修改(因为感觉有点丑...)~~ 最新版本已经支持随心所欲的滑动体验了, 也去除了滑动指示器

其他的好像没啥了...

补充一句, 该组件基本上没怎么使用太多小程序相关的特性, 所以按照这个思路用h5实现应该也是可以的, 如果有h5方面的需求应该也是可以满足的...

[drag组件地址](https://github.com/singletouch/wx-plugin/tree/master/components/drag) 