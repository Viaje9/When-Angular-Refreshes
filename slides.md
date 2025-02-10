---
theme: bricks
title: Angular 什麼時候會刷新畫面
layout: intro
mdc: true
fonts:
  sans: Chocolate Classical Sans
  serif: Chocolate Classical Sans
  mono: Chocolate Classical Sans
# transition: fade-out
---

<style>
	
	:root, html, body, pre, code {
		--prism-tab-size: 8;
		tab-size: 8;
	}
  .slidev-layout.intro h1 {
    font-size: 4rem;
    line-height: 15rem;
  }
	pre, code {
		letter-spacing: 0.1em;
	}
	span {
		tab-size: 8;
	}
</style>

# Angular 什麼時候會刷新畫面

一次帶你看懂 Angular 的變更偵測機制

<div class="absolute bottom-10">
  <span class="font-700">
    Brian 黃柏宇
  </span>
</div>

<style>
h1 {
 font-size: 20px
}
</style>

---

# Angular 是如何偵測變數異動的？

<div class="pl-10">
  <div class="mt-20 text-xl">
  其他框架偵測到變數異動的方法。
  </div>
  <div class="pl-10 mt-5 mb-10">
    <ul>
      <li>Vue2 使用 <a target="_blank" href="https://developer.mozilla.org/zh-TW/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty">Object.defineProperty</a>。</li>
      <li>Vue3 使用 <a target="_blank" href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy">Proxy Object</a>。</li>
      <li>React 使用 <a target="_blank" href="https://react.dev/reference/react/useState">useState Hook</a>。</li>
    </ul>
  </div>
  <div class="mt-10 text-xl">
    Angular 呢？它是怎麼知道變數異動了？
  </div>
</div>

---

# 同一個範例不同框架-Angular

```angular-ts {all}
import { Component } from '@angular/core';

@Component({
	selector: 'app-root',
	template: `
		<button (click)="updateValue()">點擊我更新</button>
		<p>當前值：{{ value }}</p>
	`,
})
export class UpdateValueComponent {
	value = 0;

	updateValue() {
		this.value += 1;
	}
}
```

---

# 同一個範例不同框架-Vue2

```vue {all}
<template>
	<div>
		<button @click="updateValue">點擊我更新</button>
		<p>當前值：{{ value }}</p>
	</div>
</template>

<script>
export default {
	name: 'UpdateValueComponent',
	data() {
		return {
			value: 0
		};
	},
	methods: {
		updateValue() {
			this.value += 1;
		}
	}
};
</script>
```

---

# 同一個範例不同框架-Vue3

```vue {all}
<template>
	<div>
		<button @click="updateValue">點擊我更新</button>
		<p>當前值：{{ value }}</p>
	</div>
</template>

<script setup>
import { ref } from 'vue';

const value = ref(0);

function updateValue() {
	value.value += 1;
}
</script>
```

---

# 同一個範例不同框架-React Function Component

```jsx
import React, { useState } from 'react';

function UpdateValueComponent() {
	const [value, setValue] = useState(0);

	const updateValue = () => {
		setValue(value + 1);
	};

	return (
		<div>
			<button onClick={updateValue}>點擊我更新</button>
			<p>當前值：{value}</p>
		</div>
	);
}

export default UpdateValueComponent;
```

---

# 從範例看 Angualr 與其他框架的差異

<div class="pl-3">

<table>
	<thead>
		<tr>
			<th>框架</th>
			<th>變數宣告方式</th>
			<th>更新變數方式</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>Angular</td>
			<td>Class Property</td>
			<td>直接修改 this.property</td>
		</tr>
		<tr>
			<td>Vue 2</td>
			<td>data() 返回物件</td>
			<td>直接修改 this.property</td>
		</tr>
		<tr>
			<td>Vue 3</td>
			<td>ref 或 reactive</td>
			<td>ref.value 或直接修改</td>
		</tr>
		<tr>
			<td>React</td>
			<td>useState</td>
			<td>setValue</td>
		</tr>
	</tbody>
</table>

Vue 2 、 Vue 3 看似直接修改，實則是<span class="text-red-500">宣告變數時透過 <a target="_blank" href="https://developer.mozilla.org/zh-TW/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty">Object.defineProperty</a> 和 <a target="_blank" href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy">Proxy Object</a> 包裝</span>。

不管是 Vue 還是 React ，他們在<span class="text-red-500">更新變數</span>的時候，就會去<span class="text-red-500">通知框架</span>有變數異動。
</div>

---

# 假設 Angular 是在 Click 通知框架

```angular-ts {all}
@Component({
	selector: 'app-root',
	standalone: true,
	template: `
		<button (click)="updateValue()">點擊我更新</button>
		<p>當前值：{{ value }}</p>
	`,
})
export class UpdateValueComponent {
	value = 0;

	updateValue() {
		setInterval(() => {
			this.value += 1;
		}, 1000)
	}
}
```

如果 Angular 是在 Click 的時候去通知框架要檢查，他怎麼知道 setInterval 的 callback 什麼時候回來？


---

# 研究租屋 API 的故事

<div class="pl-10">
  <div class="mt-20 text-xl">
  研究租屋 API 跟 setInterval 有什麼關係？
  </div>
  <div class="pl-7 mt-5 mb-10">
    <ol>
      <li class="mb-2">打開網頁的 Console 結果被導轉回首頁。</li>
			<li class="mb-2">研究之後推測是使用特殊技巧偵測開啟 Console 。</li>
			<li class="mb-2">嘗試複寫 setInterval 阻止偵測。</li>
    </ol>
  </div>
  <div class="mt-10 text-xl">
		<a target="_blank" href="https://www.591.com.tw/">租屋網站</a>
  </div>
</div>

---

# 複寫 setInterval 跟 Angular 偵測變數異動的關係

<div class="pl-10">
  <div class="mt-20 text-xl">
  回到一開始的問題 Angular 如何偵測變數異動？
  </div>
  <div class="pl-10 mt-5 mb-10">
    <ol class="py-5">
      <li class="py-2">假設 Angular 是在 Click 通知框架要檢查變數異動</li>
			<li class="py-2">透過複寫 setInterval 設定 callback 通知框架要檢查變數異動</li>
    </ol>
  </div>
  <div class="mt-10 text-xl">
		Angular 使用 zone.js 來處理這些非同步事件的攔截。
  </div>
</div>

--- 

# Zone.js 攔截的非同步事件

<div class="ml-10">
	<div class="my-5 text-xl">
		常見的非同步事件有哪些
	</div>
	<div class="mb-5">
		<ol>
			<li>setTimeout / setInterval</li>
			<li>requestAnimationFrame</li>
			<li>Promise</li>
			<li>DOM Event</li>
			<li>fetch / XMLHttpRequest / WebSocket</li>
		</ol>
	</div>
	<div class="mb-5">
		基本上<strong class="text-red-500">大部分</strong>非同步事件都可以透過 Zone.js 攔截。(<a href="https://github.com/angular/angular/blob/main/packages/zone.js/STANDARD-APIS.md">支援的 API</a>、<a href="https://github.com/angular/angular/blob/HEAD/packages/zone.js/NON-STANDARD-APIS.md">不支援的 API</a>)
	</div>
	<div class="mb-5">
		Zone.js 並非 Angular 獨有，也可以當作<a href="https://www.npmjs.com/package/zone.js?activeTab=readme">獨立的函式庫</a> 使用。
	</div>
	<div class="mb-5">
		NgZone 與 Zone.js 不同 ，NgZone 基於 Zone.js 再包裝。
	</div>
</div>

---

<div class="flex justify-center "> 
	<h1>點擊事件觸發畫面更新的過程</h1>
</div>


<div class="w-full flex justify-center"> 
	<img class="w-10/12" src="/img/CD01.gif" />
</div>

---

# 更新過程的各個部分

<div class="pl-10 mt-10">
  <ol>
    <li class="mb-5"><strong>Component Tree(元件樹)</strong>：隨 Angular 啟動產生元件樹，<span class="text-red-500">在當下路由的東西才會被加入進元件樹</span>。</li>
    <li class="mb-5"><strong>NgZone</strong>：最外圍的框框，負責攔截事件，通知 Angular 觸發 Change Detection。</li>
    <li class="mb-5"><strong>Dirty 標記與冒泡反應</strong>：紫色部分，點擊時會元件會被標記為 Dirty ，<span class="text-red-500">並帶有冒泡效果，向上標記為 Dirty</span>。</li>
    <li class="mb-5"><strong>Change Detection (變更檢測)</strong>：綠色部分，會從根元件遍歷元件樹，檢查元件樹是否有元件需要更新。</li>
  </ol>
</div>

---

# Default 策略 Change Detection 如何執行

<div class="ml-10">
	<div class="mb-7 mt-10 text-xl text-bold">
		Angular 的啟動的時候就會產生一顆元件樹。在 Default 會遵循以下方式執行。
	</div>
	<div class="mb-10">
		<ol>
			<li class="mb-4">整個過程從根元件開始。</li>
			<li class="mb-4">整個元件樹都會檢查是否有變更，每個節點都會被訪問。</li>
			<li class="mb-4">遍歷方向從上而下。</li>
			<li class="mb-4">節點的訪問順序遵循深度優先搜索（DFS）演算法。</li>
		</ol>
	</div>
	<div class="mb-5 text-xl">
		注意的是這裡遍歷元件樹的方式是 DFS 演算法，<a class="text-red-500" href="https://angular.love/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article" target="_blank">這跟觸發 ngDoCheck 的時機不同</a>。
	</div>
</div>

---

# 更新過程的幾個問題

<div class="ml-8 mt-5">
	<div class="mb-10">
		<ul>
			<li class="mb-3">
				<span class="text-lg text-bold text-lime-800">紫色的 Dirty 做什麼用的，為什麼需要冒泡反應？</span>
				<ol class="mt-3" v-click>
					<li class="mb-3 text-base">
						Dirty 僅在 OnPush 有用，當元件為 OnPush 且被標記為 Dirty 才會觸發 Change Detection。
					</li>
					<li class="mb-3 text-base">
						OnPush 僅被標記 Dirty 才會進行檢查，如果父元件沒被標記，那麼點擊觸的那個元件不會被檢查到，所以才需要冒泡反應。
					</li>
				</ol>
			</li>
			<li class="mb-3">
				<span class="text-lg text-bold text-lime-800">如果使用 setInterval 是不是會一直觸發 Change Detection？</span>
				<ol class="mt-3" v-click>
					<li class="mb-3 text-base">
						沒錯使用 setInterval 他就會一直觸發 Change Detection ，所以可以透過 OnPush 或是 NgZone.runOutsideAngular 避免。
					</li>
				</ol>
			</li>
			<li class="mb-3">
				<span class="text-lg text-bold text-lime-800">為什麼 Change Detection 要檢查整個元件樹，不能檢查點擊觸發的元件就好嗎？</span>
				<ol class="mt-3" v-click>
					<li class="mb-3 text-base">
						因為有些事件並不是透過點擊，所以無法得知是哪個元件的變數有異動，所以直接檢查整個元件樹。
					</li>
					<li class="mb-3 text-base">
						或是使用 OnPush 策略，可以檢查跳過某些元件節點不檢查。
					</li>
				</ol>
			</li>
		</ul>
	</div>
</div>

---

<div class="flex justify-center "> 
	<h1>setInterval 觸發畫面更新的過程</h1>
</div>


<div class="w-full flex justify-center"> 
	<img class="w-10/12" src="/img/CD02.gif" />
</div>

---

<div class="flex justify-center "> 
	<h1>使用 OnPush 時點擊事件觸發畫面更新的過程</h1>
</div>


<div class="w-full flex justify-center"> 
	<img class="w-10/12" src="/img/CD03.gif" />
</div>

---

# 會觸發 Dirty 標記的情境

<div class="ml-12">
	<div class="mb-5 mt-7 text-xl text-bold">
		什麼會讓元件變得「dirty」
	</div>
	<div class="mb-5">
		<ol class="list-decimal ml-5">
			<li class="mb-4">
        <span class="font-bold">@Input 變更</span> ( <span class="text-red-500">immutable</span> )。
      </li>
			<li class="mb-4">
        <span class="font-bold">Template 綁定的事件觸發</span> (包括 <code class="inline-block px-1 bg-gray-200 rounded">Output()</code> emit 和 <code class="inline-block px-1 bg-gray-200 rounded">HostListener</code>)。
      </li>
			<li class="mb-4">使用 <code class="inline-block px-1 bg-gray-200 rounded">AsyncPipe</code>接收到新值。
      </li>
			<li class="mb-4">
        直接呼叫 <code class="inline-block px-1 bg-gray-200 rounded">ChangeDetectorRef.markForCheck()</code>。
      </li>
			<li class="mb-4">
        <code class="inline-block px-1 bg-gray-200 rounded">@defer</code> 區塊的狀態變化。
      </li>
		</ol>
	</div>
	<div class="mb-5 text-xl">
		要記住這些事件會<span class="text-red-500">「冒泡」</span>，意思是如果<span class="text-red-500">子元件</span>被標記為「dirty」，他的<span class="text-red-500">父元件</span>也會被標記為「dirty」，並且<span class="text-red-500">一路到根元件</span>而會觸發這個行為的方法叫 <code class="inline-block px-1 bg-gray-200 rounded">markViewDirty()</code>。
	</div>
</div>

---

# 複習 ChangeDetectorRef

<div class="ml-10 mt-15">
	<div class="mb-5">
		<ol>
			<li class="mb-4">
				<code class="inline-block px-1 bg-gray-200 rounded">detach()</code> -
				將元件的 ChangeDetection 暫停，<strong class="text-red-500">即使標記為
				Dirty 也不會更新</strong>，其子元件也是。
			</li>
			<li class="mb-4">
				<code class="inline-block px-1 bg-gray-200 rounded">reattach()</code> -
				恢復元件的 ChangeDetection 。
			</li>
			<li class="mb-4">
				<code class="inline-block px-1 bg-gray-200 rounded">markForCheck()</code>
				- 將該元件標記為 Dirty 以及<strong class="text-red-500">
				向上遍歷所有祖先元件並標記，等待下一個
				ChangeDetection 週期更新標記的元件。</strong>
			</li>
			<li class="mb-4">
				<code class="inline-block px-1 bg-gray-200 rounded">detectChanges()</code>
				-
				<strong class="text-red-500">立即更新當前元件</strong>以及向下檢查其子元件是否為
				Dirty。
			</li>
		</ol>
	</div>
	<div class="mb-5 text-xl">
		正常情況下只有在
		<code class="inline-block px-1 bg-gray-200 rounded">OnPush</code>
		策略會需要使用
		<code class="inline-block px-1 bg-gray-200 rounded">ChangeDetectorRef</code>
		。
	</div>
</div>

---

# NgZone 是如何通知 Angular 要觸發檢查機制的？


<div class="">
	<div class="mb-7 mt-10 text-lg text-bold">
		NgZone 的自定義事件 onMicrotaskEmpty
	</div>
	<div class="mb-5 text-lg">
		Zone 跟 NgZone 還是有些區別，像是有自定義的事件 <code class="inline-block px-1 bg-gray-200 rounded">onMicrotaskEmpty</code>，用於判斷 Microtask 是否空了，如果空了會去執行 <code class="inline-block px-1 bg-gray-200 rounded">ApplicationRef.tick()</code> 觸發 Change Detection 。
	</div>
</div>

```ts {all}
export class ApplicationRef {
	constructor() {
		this._onMicrotaskEmptySubscription = this._zone.onMicrotaskEmpty.subscribe({
			next: () => {
				this._zone.run(() => {
					this.tick();
				});
			},
		});
	}
}
```

---

# ApplicationRef.tick 會做什麼？

<div class="">
	<div class="my-5 text-lg text-bold">
		從程式碼中，可能會注意到一個熟悉的詞彙：<code class="inline-block px-1 bg-gray-200 rounded">detectChanges</code>。這實際上就是 <code class="inline-block px-1 bg-gray-200 rounded">ChangeDetectorRef.detectChanges()</code>。此處的 <code class="inline-block px-1 bg-gray-200 rounded">view</code> 指的是根元件，換句話說，<code class="inline-block px-1 bg-gray-200 rounded">tick</code> 實際上是在根元件上執行 <code class="inline-block px-1 bg-gray-200 rounded">detectChanges()</code>。
	</div>
</div>
```ts {all}
export class ApplicationRef {
	tick(): void {
		try {
			this._runningTick = true;
			for (let view of this._views) {
				view.detectChanges();
			}
			if (typeof ngDevMode === 'undefined' || ngDevMode) {
				for (let view of this._views) {
					view.checkNoChanges();
				}
			}
		} catch (e) { ... } finally { ... }
	}
}
```

---

# 觸發 ChangeDetection 一定要有 Zone 嗎？

<div class="ml-8 mt-10">
  <div class="mb-10">
    <ul>
      <li class="mb-10">
        <span class="text-xl text-bold text-lime-800">NgZone 的通知機制</span>
        <ul class="mt-3" v-click>
					<span class="text-lg text-bold">
						NgZone 透過<strong class="text-red-500">攔截非同步事件</strong>，在事件完成後觸發 <code class="inline-block px-1 bg-gray-200 rounded">onMicrotaskEmpty</code>，<strong class="text-red-500">通知 Angular 進行變更檢查</strong>。
					</span>
        </ul>
      </li>
      <li class="mb-10">
        <span class="text-xl text-bold text-lime-800">實際觸發 Change Detection 的關鍵</span>
        <div class="mt-3" v-click>
          <span class="text-lg text-bold">Change Detection 執行<strong class="text-red-500">並不依賴 NgZone</strong>，實際上是透過以下方式<strong class="text-red-500">手動觸發</strong>：</span>
          <ul class="mt-3 ml-5 list-disc">
            <li class="mb-3 text-base">
              使用 <code class="inline-block px-1 bg-gray-200 rounded">ChangeDetectorRef.detectChanges()</code>
            </li>
            <li class="mb-3 text-base">
              呼叫 <code class="inline-block px-1 bg-gray-200 rounded">ApplicationRef.tick()</code>
            </li>
          </ul>
        </div>
      </li>
    </ul>
  </div>
</div>

---

# Angular Signals 

```angular-ts {all}
import { Component, signal } from '@angular/core';

@Component({
	selector: 'app-root',
	template: `
		<button (click)="updateValue()">點擊我更新</button>
		<p>當前值：{{ value() }}</p>
	`,
})
export class UpdateValueComponent {
	value = signal(0);

	updateValue() {
		this.value.update(v => v + 1);
	}
}
```

--- 

<div class="flex justify-center "> 
	<h1>Signals 理想情況的 ChangeDetection 運作方式</h1>
</div>


<div class="w-full flex justify-center"> 
	<img class="w-10/12" src="/img/CD04.gif" />
</div>

---

# Signals 情境下的 ChangeDetection：理想情況

```angular-ts {all}
import { Component, signal, ChangeDetectionStrategy } from '@angular/core';

@Component({
	selector: 'app-root',
	changeDetection: ChangeDetectionStrategy.OnPush,
	template: `
		<h1> {{ title() }} </h1>
	`,
})
export class HomeComponent {
	title = signal('Angular Signals');

	ngOnInit() {
		setTimeout(() => {
			this.title.update('Hello Angular Signals');
		}, 2000);
	}
}
```

--- 

# Signals 為 ChangeDetection 所帶來的改變

<div class="pl-10 mt-5">
	<ol class="list-decimal ml-5">
		<li class="mb-4">
			<div class="font-bold text-lg">Consumer Dirty 標記</div>
			<div class="mt-1 text-base">
				當 Template 中的 <code class="inline-block px-1 bg-gray-200 rounded">signal()</code> 偵測到數值變動時，<strong
					class="text-red-500"> Signal 會在內部</strong>標記為 Consumer Dirty。
			</div>
		</li>
		<li class="mb-4">
			<div class="font-bold text-lg">HasChildViewsToRefresh 標記</div>
			<div class="mt-1 text-base">
				當元件被標記為 <strong>Consumer Dirty</strong> 時，Angular 會呼叫 <code
						class="inline-block px-1 bg-gray-200 rounded">markAncestorsForTraversal()</code> 他會將其父元件標記為 <strong>HasChildViewsToRefresh</strong>，此行為會<strong class="text-red-500">類似於傳統 Change Detection 的 Dirty 冒泡</strong>。
			</div>
		</li>
		<li class="mb-4">
			<div class="font-bold text-lg">更聰明的 Change Detection</div>
			<div class="mt-1 text-base">
				Change Detection 執行時，遇到被標記為 <strong>HasChildViewsToRefresh</strong> 的元件時，<strong
					class="text-red-500">不會深入檢查其子元件</strong>，而是<strong class="text-red-500">直接跳過</strong>。只有遇到 <strong>Consumer Dirty</strong> 的元件時，才會進行更新。
			</div>
		</li>
		<li class="mb-4">
			<div class="font-bold text-lg">擺脫 <code class="inline-block px-1 bg-gray-200 rounded">NgZone</code>
				的依賴</div>
			<div class="mt-1 text-base">
				Signals 的變更偵測機制<strong class="text-red-500">不再需要依賴 NgZone 來通知框架觸發檢查</strong>，而是採用更<strong
					class="text-red-500">直接、高效</strong>的方式。
			</div>
		</li>
	</ol>
</div>

---

<div class="flex justify-center "> 
	<h1>如果是透過點擊來變更 Signals 的值</h1>
</div>


<div class="w-full flex justify-center"> 
	<img class="w-10/12" src="/img/CD05.gif" />
</div>

--- 

# 新舊機制混合的 Change Detection

<div class="ml-5">
	<div class=" text-lg text-bold ">
		點擊事件觸發 Signals 變更：<strong class="text-red-500">仍有「冒泡」現象</strong>
	</div>
	<div class="ml-5">
		<p class="text-base">
			當透過 <strong class="text-red-500">點擊事件</strong> 來更新 Signals 的值時，由於 Angular <strong
				class="text-red-500">舊有的變更偵測規則仍然生效</strong>，因此 <strong class="text-red-500">「冒泡」行為依然存在</strong>。
		</p>
		<p class="text-base mt-2">
			在這種情況下，即使使用了 Signals，其變更偵測的行為，與原本 <code class="inline-block px-1 bg-gray-200 rounded">OnPush</code> 策略相比，<strong
				class="text-red-500">在效能上的差異並不明顯</strong>。
		</p>
	</div>
	<div class=" text-lg text-bold ">
		Signals 真正優勢：<strong class="">避免非同步操作的「冒泡」</strong>
	</div>
	<div class="ml-5">
		<p class="text-base">
			Signals 的優勢在於，對於以下這些 <strong class="text-red-500">非同步操作</strong> 觸發的變更，<strong
				class="text-red-500">不會產生「冒泡」現象</strong>，從而實現更精準、更高效的變更偵測：
		</p>
	</div>
	<div class="ml-5">
		<ol class="list-disc">
			<li class="mb-2 text-base"><code class="inline-block px-1 bg-gray-200 rounded">setInterval</code>, <code
					class="inline-block px-1 bg-gray-200 rounded">setTimeout</code></li>
			<li class="mb-2 text-base"><code class="inline-block px-1 bg-gray-200 rounded">Observable.subscribe</code>, <code
					class="inline-block px-1 bg-gray-200 rounded">toSignal(Observable)</code> (例如 <code
					class="inline-block px-1 bg-gray-200 rounded">HttpClient</code> call)</li>
			<li class="mb-2 text-base"><code class="inline-block px-1 bg-gray-200 rounded">RxJS fromEvent</code>, <code
					class="inline-block px-1 bg-gray-200 rounded">Renderer2.listen</code></li>
		</ol>
	</div>
</div>

---

<div class="flex justify-center "> 
	<h1>沒有 Zone 的 Angular </h1>
</div>


<div class="w-full flex justify-center"> 
	<img class="w-10/12" src="/img/CD06.gif" />
</div>

--- 

<div>
		<h1>什麼情況 Notify 會被呼叫</h1>
</div>
<div class="pl-10">
	<div class="text-xl text-bold">
		Notify 在以下 <strong class="text-red-500">三個主要情境</strong> 下被觸發
	</div>
	<div class="mb-5">
		<ol class="list-decimal ml-5">
			<li class="mb-3">
				<span class="text-lg text-bold text-yellow-900">Template 讀取 Signals 值</span>
				<div class="mt-1 text-sm">
					當 <code class="inline-block px-1 bg-gray-200 rounded">signal()</code> 的值在 Template 中被讀取時 (例如
					<code>{ { value() } }</code> )，呼叫 <code
						class="inline-block px-1 bg-gray-200 rounded">markAncestorsForTraversal()</code> 時。
				</div>
			</li>
			<li class="mb-3">
				<span class="text-lg text-bold text-yellow-900">元件被標記為 Dirty</span>
				<div class="mt-1 text-sm">
					當元件透過 <code class="inline-block px-1 bg-gray-200 rounded">markViewDirty()</code> 函數被標記為 Dirty 時，這通常發生在：
					<ul class="list-disc ml-8 mt-1">
						<li class="text-xs">
							<span class="font-bold">@Input 異動</span> ( <span class="text-red-500">immutable</span> )。
						</li>
						<li class="text-xs">
							<span class="font-bold">Template 綁定的事件觸發</span> (包括 <code class="inline-block px-1 bg-gray-200 rounded">Output()</code> emit 和 <code class="inline-block px-1 bg-gray-200 rounded">HostListener</code>)。
						</li>
						<li class="text-xs">使用 <code class="inline-block px-1 bg-gray-200 rounded">AsyncPipe</code>接收到新值。
						</li>
						<li class="text-xs">
							直接呼叫 <code class="inline-block px-1 bg-gray-200 rounded">ChangeDetectorRef.markForCheck()</code>。
						</li>
						<li class="text-xs">
							<code class="inline-block px-1 bg-gray-200 rounded">@defer</code> 區塊的狀態變化。
						</li>
					</ul>
				</div>
			</li>
			<li class="mb-3">
				<span class="text-lg text-bold text-yellow-900">AfterRender Hooks 與 View 生命周期</span>
				<div class="mt-1 text-sm">
					當 <code class="inline-block px-1 bg-gray-200 rounded">afterRender</code> hook 被註冊、View 重新附加到 Change Detection
					Tree 或從 DOM 移除時。
					<div class="mt-1">
						<strong class="text-red-500">注意：</strong> 此情境下 <code
							class="inline-block px-1 bg-gray-200 rounded">notify</code> 僅執行 hooks，<strong class="text-red-500">不會刷新
							View</strong>。
					</div>
				</div>
			</li>
		</ol>
	</div>
</div>


---

# Standalone 如何不使用 Zone 

```ts {all}
import { ApplicationConfig, provideExperimentalZonelessChangeDetection } from '@angular/core';
import { bootstrapApplication } from "@angular/platform-browser";
import { AppComponent } from "./app.component";

export const appConfig: ApplicationConfig = {
	providers: [
		provideExperimentalZonelessChangeDetection(), // 新增 Zoneless provider
	]
};

bootstrapApplication(AppComponent, appConfig)
	.catch((err) => console.error(err));

// 從 angular.json 中移除以下內容：
//
// "polyfills": [
//   "zone.js"
// ],
```

---

# Angular Zoneless 的優點 

<div class="pl-10 mt-10">
	<div class="mb-5 mt-10 text-xl text-bold">
		Zoneless 是什麼？
	</div>
	<div class="ml-5 mb-5 text-xl">
		Angular Zoneless 指的是在 Angular 應用中<strong class="text-red-500">移除 Zone.js</strong>，改用 Notify 觸發 Change Detection。
	</div>
	<div class="mb-5 text-xl text-bold">
		優點
	</div>
	<div class="mb-5">
		<ol class="list-disc ml-5">
			<li class="mb-4">
				<strong class="">減少 Bundle Size：</strong> 移除 Zone.js 可以<strong class="text-red-500">縮減應用程式的
					bundle 大小</strong>，減少下載時間。
			</li>
			<li class="mb-4">
				<strong class="">潛在的效能提升：</strong> 減少 Zone.js 的額外開銷，理論上可能提升效能 (實際效果需評估)。
			</li>
			<li class="mb-4">
				<strong class="">保持更好的相容性：</strong> Zone.js 可能影響部分 Library 的相容性，Angular Zoneless 則提升整合性與新技術適配性。
			</li>
		</ol>
	</div>
</div>

---

# Angular Zoneless 的缺點與注意事項

<div class="pl-10 mt-10">
	<div class="mb-5 text-xl text-bold">
		缺點與注意事項
	</div>
	<div class="mb-5">
		<ol class="list-disc ml-5">
			<li class="mb-3">
				<strong class="text-red-500">學習成本：</strong> 已習慣 Zone.js 的開發者需要適應新方式，但對新手或來自其他框架的人影響較小。
			</li>
			<li class="mb-3">
				<strong class="text-red-500">實驗階段：</strong> Angular Zoneless 目前仍處於<strong class="text-red-500">實驗性階段
					(experimental)</strong>，穩定性和社群支援可能不如傳統模式。
			</li>
			<li class="mb-3">
				<strong class="text-red-500">遷移風險：</strong> 現有應用程式遷移到 Angular Zoneless 可能需要較大的改動和測試。
			</li>
		</ol>
	</div>
	<div class="mb-5 text-xl">
		<strong class="text-lime-800">總結：</strong> Angular Zoneless 提供了一種新的選擇，但在正式環境使用前，務必仔細評估其優缺點和潛在風險。
	</div>
</div>

---

# 今天提到的內容

<div class="mt-7">
	<div class="mb-5">
		<ol class="">
			<li class="mb-4">
				<span class="font-bold">Angular 如何判斷執行變更檢測：</span>
				<span class="text-base">比較 Angular 與 Vue、React 等框架在變更偵測上的差異。</span>
			</li>
			<li class="mb-4">
				<span class="font-bold">Zone.js 的關鍵作用：</span>
				<span class="text-base"> Zone.js 如何攔截非同步事件，並觸發 Angular 的畫面更新。</span>
			</li>
			<li class="mb-4">
				<span class="font-bold">Default 與 OnPush 變更偵測策略：</span>
				<span class="text-base">掌握 Angular 預設的變更檢測方式，以及 OnPush 策略如何優化效能。</span>
			</li>
			<li class="mb-4">
				<span class="font-bold">NgZone 的幕後運作：</span>
				<span class="text-base">理解 NgZone 如何透過 <code>onMicrotaskEmpty</code> 事件通知 Angular 進行變更檢查。</span>
			</li>
			<li class="mb-4">
				<span class="font-bold">Signals 如何革新變更偵測：</span>
				<span class="text-base">探索 Signals 如何帶來更精準、高效的變更偵測，以及擺脫對 Zone.js 的依賴。</span>
			</li>
			<li class="mb-4">
				<span class="font-bold">Angular Zoneless 的優缺點：</span>
				<span class="text-base">評估在 Angular 應用中移除 Zone.js 的優勢與潛在風險。</span>
			</li>
		</ol>
	</div>
</div>

---

# 結語

<div class="pl-5 mt-10">
	<ul class="list-disc ml-5">
		<li class="mb-4">
			<strong>從 Change Detection 的角度回顧變遷：</strong> 
			今天帶大家一覽傳統的 Change Detection，並深入探討 Signals 如何改變現有機制。
		</li>
		<li class="mb-4">
			<strong>Angular 正處於過渡時期：</strong> 
			目前 Angular 還未完全擺脫 Zone.js，正在逐步邁向 Zoneless 方向。
		</li>
		<li class="mb-4">
			<strong>轉變過程帶來的挑戰：</strong> 
			在邁向 Zoneless 的過程中，開發者可能會面臨一定的衝擊與適應期。
		</li>
		<li class="mb-4">
			<strong>掌握變革，應對未來：</strong> 
			希望這場分享能幫助大家理解 Angular 為何要進行這樣的轉變，當變革來臨時，我們能夠更清楚它的運作原理。
		</li>
	</ul>
</div>

---

# 附錄

<div class="pl-10">
	<ol>
		<li class="mb-2 text-base">
			<a target="_blank" href="https://angular.love/the-latest-in-angular-change-detection-zoneless-signals" class="text-blue-500">The
				Latest in Angular Change Detection</a>
			<span> - 貫穿全文的主題，內容不理解的部分可以參考這裡。</span>
		</li>
		<li class="mb-2 text-base">
			<a target="_blank" href="https://itnext.io/a-change-detection-zone-js-zoneless-local-change-detection-and-signals-story-9344079c3b9d"
				class="text-blue-500">A change detection, zone.js, zoneless, local change detection, and signals story 📚</a>
			<span> - 這篇更深入的探討簡報的主題 ，而且講的很完整。</span>
		</li>
		<li class="mb-2 text-base">
			<a target="_blank" href="https://angular.love/everything-you-need-to-know-about-change-detection-in-angular"
				class="text-blue-500">Everything you need to know about change detection in Angular</a>
			<span> - 更了解 Change Detection 可以看這篇。</span>
		</li>
		<li class="mb-2 text-base">
			<a target="_blank" href="https://angular.love/running-change-detection-manual-control" class="text-blue-500">Running change
				detection – Manual control</a>
			<span> - 想更手動控制 Change Detection 的可以參考這篇。</span>
		</li>
		<li class="mb-2 text-base">
			<a target="_blank" href="https://angular.love/do-you-still-think-that-ngzone-zone-js-is-required-for-change-detection-in-angular"
				class="text-blue-500">Do you still think that NgZone (zone.js) is required for change detection in Angular?</a>
			<span> - 這篇解釋為什麼不需要 NgZone 也可以觸發 Change Detection。</span>
		</li>
		<li class="mb-2 text-base">
			<a target="_blank" href="https://wazeedmohammad.medium.com/zone-js-ngzone-and-applicationref-in-angular-87a9a6a95f4a"
				class="text-blue-500">zone.js, NgZone, and ApplicationRef in Angular</a>
			<span> - 初入 NgZone 看這裡。</span>
		</li>
		<li class="mb-2 text-base">
			<a target="_blank" href="https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found"
				class="text-blue-500">I reverse-engineered Zones (zone.js) and here is what I’ve found</a>
			<span> - 深入 Zone 看這裡。</span>
		</li>
		<li class="mb-2 text-base">
			<a target="_blank" href="https://angular.love/signals-in-angular-deep-dive-for-busy-developers#Signals%20as%20primitives"
				class="text-blue-500">Signals in Angular: deep dive for busy developers</a>
			<span> - 深入 Signals 看這裡。</span>
		</li>
	</ol>
</div>


