# 程式碼背後的魔法 – Angular 如何更新畫面？

作為一位前端開發者，我們每天都在與框架打交道，操控著瀏覽器畫面上的一切。當我們修改變數，觸發事件，框架彷彿擁有魔法一般，能即時更新畫面，呈現最新的狀態。你是否曾好奇，這背後究竟是什麼樣的機制在運作？特別是對於 Angular 而言，它又是如何偵測到變數的變化，並驅動畫面更新的呢？

與 Vue 2、Vue 3 和 React 等框架不同，Angular 在變數更新的方式上，似乎顯得格外「直接」。Vue 透過 `Object.defineProperty` 或 `Proxy Object`，React 則依賴 `useState` Hook，它們都在變數宣告階段就埋下了監聽的種子。而 Angular，我們似乎可以透過直接修改 Class 的屬性，畫面就能自動更新，這背後究竟有什麼奧秘？

本文將帶你踏上一段探索 Angular 變更偵測機制的旅程，從最基礎的概念開始，逐步深入到 Zone.js、Signals，乃至於最新的 Zoneless 技術，揭開 Angular 畫面更新背後的層層面紗。

## 範例程式碼：跨框架的比較與 Angular 的「直接」

為了更直觀地理解不同框架在變更偵測上的差異，我們先來看一個簡單的範例。這個範例的功能很單純：當使用者點擊按鈕時，畫面上的數值會遞增。我們將分別使用 Angular、Vue 2、Vue 3 和 React 來實現這個功能，並比較它們的程式碼：

```ts
// Angular
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


```html
<!-- Vue2 -->
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

```html
<!-- Vue3 -->
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

```jsx
// React Function Component
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

觀察這些範例程式碼，你會發現一個有趣的現象：Angular 在更新變數 `value` 時，使用的是最直接的方式 `this.value += 1`。相較之下，Vue 3 和 React 都需要透過特定的 API (`ref.value += 1` 或 `setValue(value + 1)`) 來觸發更新。Vue 2 雖然也看似直接修改 `this.value`，但實際上 Vue 內部透過 `Object.defineProperty` 對 `data` 中的屬性進行了包裝，使其具備了響應式能力。

| 框架    | 變數宣告方式     | 更新變數方式                         |
| :------ | :--------------- | :----------------------------------- |
| Angular | Class Property   | 直接修改 `this.property`              |
| Vue 2   | `data()` 返回物件 | 直接修改 `this.property`              |
| Vue 3   | `ref` 或 `reactive` | `ref.value` 或直接修改              |
| React   | `useState`       | `setValue`                           |

這種「直接」的方式，不禁讓人好奇：Angular 究竟是如何感知到變數的變化的？難道它真的擁有魔法，能夠自動追蹤所有變數的修改嗎？

## 從 Click 事件到 `setInterval`：初步猜想與疑點

一個初步的猜想是，Angular 的變更偵測可能是由事件驅動的。例如，當使用者點擊按鈕時，Angular 能夠偵測到 `(click)` 事件的發生，並在這個事件處理函數執行完畢後，觸發變更偵測，更新畫面。

這個猜想看似合理，但如果我們考慮到非同步操作，就會發現問題所在。假設我們將數值更新的邏輯放在 `setInterval` 中執行：

```ts
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

在這個例子中，變數的更新並不是由 `(click)` 事件直接觸發的，而是由 `setInterval` 定時 callback 觸發的。如果 Angular 的變更偵測僅僅依賴於事件驅動，那麼它如何知道 `setInterval` 的 callback 何時執行，並在適當的時機觸發變更偵測呢？

這個問題引導我們走向 Angular 變更偵測機制的核心 – Zone.js。

## 幕後功臣：Zone.js – 攔截非同步的守護者

為了揭開 Angular 變更偵測的神秘面紗，我們不得不提到一個幕後功臣 – **Zone.js**。

你或許對 Zone.js 這個名字感到陌生又熟悉，但它卻在 Angular 應用程式中扮演著至關重要的角色。為了更好地理解 Zone.js 的作用，我想先分享一個我個人的經歷。

前陣子，我在研究某個租屋網站的 API 時，遇到了一個非常有趣的現象。當我打開瀏覽器的開發者工具 (Console) 時，網頁竟然會自動跳轉回首頁或是直接關閉分頁。經過一番研究，我推測這個網站使用了某種特殊的技巧來偵測開發者工具是否被開啟，以防止使用者直接在瀏覽器中查看 API 請求。

為了繞過這種偵測機制，我嘗試複寫了 JavaScript 的 `setInterval` 函數，希望能阻止網站的偵測。沒想到，這個看似無關的舉動，讓我聯想到了 `setInterval` 與 Angular 變更偵測機制之間的關聯。

事實上，Angular 正是透過 Zone.js 來**攔截和管理各種非同步操作**，其中包括：

*   `setTimeout` / `setInterval` (定時器)
*   `requestAnimationFrame` (動畫幀)
*   `Promise` (Promise 物件)
*   DOM Events (例如 `click`, `input`, `mouseover` 等用戶互動事件)
*   網路請求相關 API (`fetch`, `XMLHttpRequest`, `WebSocket`)

Zone.js 的作用，就像是一位默默守護的間諜，它無聲無息地監控著應用程式中各種非同步事件的發生。當這些非同步事件完成時，Zone.js 就會及時通知 Angular，觸發變更偵測，進而更新畫面。

值得注意的是，**Zone.js 並非 Angular 專屬的技術**，它本身是一個獨立的 JavaScript 函式庫，可以用於任何需要追蹤非同步操作的場景。在 Angular 中，我們使用的其實是 **NgZone**，它是基於 Zone.js 進行了進一步封裝，更貼合 Angular 的變更偵測機制。

## 視覺化變更流程：Default 策略下的動畫解析

為了更形象地理解 Angular 在 Default 策略下的變更偵測流程，讓我們借助一張 GIF 動畫來進行解析：

![](/public/img/CD01.gif)
⬆️ (CD01.gif)

這張動畫生動地展示了當使用者在 Angular 應用程式中觸發點擊事件後，畫面更新的完整過程。讓我們逐一拆解動畫中的關鍵元素：

1.  **Component Tree (元件樹)：** Angular 應用程式在啟動時，會根據元件的結構，建立一棵元件樹。這棵樹狀結構代表了應用程式的 UI 組成，只有當前路由下的元件才會被納入元件樹中。

2.  **NgZone：** 動畫中最外層的圓形邊界，代表著 NgZone 的作用範圍。NgZone 負責攔截應用程式中發生的各種非同步事件，並在事件完成後，通知 Angular 觸發變更偵測。

3.  **Dirty 標記與冒泡：** 動畫中紫色的圓點代表被標記為 "Dirty" 的元件。當點擊事件發生在某個元件上時，這個元件首先會被標記為 Dirty。更重要的是，Dirty 標記會產生**冒泡效應**，向上傳遞到父元件，直到根元件。這種冒泡機制確保了即使子元件的數據發生變化，也能觸發整個元件樹的變更檢測。

4.  **Change Detection (變更檢測)：** 動畫中綠色的掃描線代表變更檢測的過程。Angular 的變更檢測會從根元件開始，**深度優先遍歷 (DFS, Depth-First Search) 整個元件樹**，檢查每個元件是否需要更新。

在 Default 變更偵測策略下，Angular 會遵循以下步驟執行變更檢測：

1.  **從根元件開始：** 變更檢測總是從應用程式的根元件開始。
2.  **檢查整個元件樹：** 整個元件樹中的每個元件節點都會被訪問和檢查，無論元件的數據是否真的發生了變化。
3.  **自上而下的遍歷方向：** 變更檢測的遍歷方向是自上而下的，從父元件到子元件。
4.  **深度優先搜索 (DFS) 演算法：** 元件樹的訪問順序遵循深度優先搜索演算法。這意味著變更檢測會優先深入到元件樹的深層節點，再回溯到兄弟節點。

理解 Default 策略下的變更檢測流程至關重要，這不僅是 Angular 變更檢測的基礎，也為我們後續理解 OnPush 策略和 Signals 的變更偵測機制奠定了基礎。

## 釐清常見疑問：深入理解變更偵測的細節

在理解了 Default 策略下的變更檢測流程後，你可能會有一些疑問。接下來，我們將針對一些常見的疑問進行解答，幫助你更深入地理解 Angular 的變更偵測機制：

*   **紫色的 Dirty 標記究竟有什麼作用？為什麼需要冒泡反應？**

    Dirty 標記在 Default 策略下看似沒有明顯的作用，但它在 **OnPush 變更偵測策略**中扮演著至關重要的角色。當元件的變更偵測策略設定為 `OnPush` 時，只有當元件被標記為 Dirty 時，Angular 才會對該元件及其子元件進行變更檢測。

    而 Dirty 標記的冒泡反應，則是為了確保 `OnPush` 策略能夠正常運作。在 `OnPush` 策略下，如果父元件沒有被標記為 Dirty，即使子元件的數據發生了變化，Angular 也**不會**檢查子元件。透過 Dirty 標記的冒泡機制，當子元件被標記為 Dirty 時，其父元件也會被標記，從而確保了變更檢測能夠正確地觸發到需要更新的元件。

*   **如果我在程式碼中使用了 `setInterval`，是否會一直觸發 Change Detection？**

    是的，如果你在 Angular 元件中使用了 `setInterval`，並且在 `setInterval` 的回調函數中修改了元件的數據，那麼 `setInterval` 就會**持續不斷地觸發 Change Detection**。這是因為 `setInterval` 是一種非同步操作，每次 `setInterval` 的回調函數執行時，Zone.js 都會捕獲到這個事件，並通知 Angular 觸發變更檢測。

    這種行為在某些場景下可能是預期的，但在許多情況下，過於頻繁的變更檢測會導致效能問題。為了避免 `setInterval` 過度觸發變更檢測，你可以考慮使用以下方法：

    *   **OnPush 變更偵測策略：** 將元件的變更偵測策略設定為 `OnPush`，並結合 `@Input` 和 Signals 等技術，更精細地控制變更檢測的觸發時機。
    *   **`NgZone.runOutsideAngular`：** 將 `setInterval` 的程式碼放在 `NgZone.runOutsideAngular` 區塊中執行，使其脫離 NgZone 的監控範圍，從而避免觸發變更檢測。

*   **為什麼 Change Detection 要檢查整個元件樹，而不能只檢查點擊事件觸發的元件？**

    Default 策略下的變更檢測之所以需要檢查整個元件樹，主要是因為 Angular 的變更偵測機制是**基於數據變更**的，而不是基於事件觸發的元件。在許多情況下，數據的變更可能並不是由用戶的直接點擊事件觸發的，例如：

    *   來自後端服務器的數據更新 (透過 `HttpClient` 或 WebSocket 等 API 獲取)
    *   定時器觸發的數據更新 (`setInterval`, `setTimeout`)
    *   其他元件或服務觸發的數據更新

    在這些非直接事件觸發的場景下，Angular 無法預先判斷是哪個元件的數據發生了變化，因此只能透過檢查整個元件樹，確保所有數據變化都能被檢測到，並更新到畫面。

    當然，檢查整個元件樹會帶來一定的效能開銷。為了優化效能，Angular 提供了 **OnPush 變更偵測策略**。`OnPush` 策略允許我們更精細地控制變更檢測的範圍，跳過那些數據沒有發生變化的元件節點，從而提升應用程式的效能。

## 不同情境下的變更流程：動畫輔助深入解析

為了更深入地理解不同情境下的變更偵測流程，我們再次借助 GIF 動畫 (CD02.gif, CD03.gif) 進行視覺化解析：

![](/public/img/CD02.gif)
⬆️ (CD02.gif)

![](/public/img/CD03.gif)
⬆️ (CD03.gif)

*   **`setInterval` 觸發畫面更新的過程 (CD02.gif)：** 這個動畫展示了當使用 `setInterval` 定時更新數據時，變更檢測的觸發流程。你會看到，即使沒有用戶互動事件，`setInterval` 仍然會持續觸發變更檢測，導致整個元件樹被重複檢查。

*   **使用 `OnPush` 時點擊事件觸發畫面更新的過程 (CD03.gif)：** 這個動畫展示了當元件使用 `OnPush` 策略時，點擊事件觸發的變更流程。你會發現，在 `OnPush` 策略下，只有被標記為 Dirty 的元件及其父元件才會被檢查，變更檢測的範圍大大縮小，效能也得到了提升。

透過這兩張動畫的對比，我們可以更直觀地理解不同情境下變更檢測的差異，以及 `OnPush` 策略在優化效能方面的作用。

## Dirty 標記觸發條件：全面掌握變更偵測的觸發點

現在我們已經知道 Dirty 標記在變更偵測中扮演著重要的角色，那麼，究竟有哪些情況會讓元件被標記為 "dirty" 呢？以下列舉了幾種常見的觸發 Dirty 標記的情境：

1.  **`@Input` 變更 (Immutable)：** 當元件的 `@Input` 屬性值發生變化時，Angular 會將該元件標記為 Dirty。需要注意的是，這裡的變更偵測是基於 **Immutable (不可變) 的比較**。也就是說，只有當 `@Input` 屬性指向一個新的物件引用時，Angular 才會認為 `@Input` 發生了變化。如果 `@Input` 屬性指向的物件內部屬性發生了變化，但物件引用沒有改變，Angular **不會** 檢測到這個變化。

2.  **Template 綁定的事件觸發：** 當元件 Template 中綁定的事件 (例如 `(click)`, `(input)`, `(mouseover)` 等) 被觸發時，Angular 會將該元件標記為 Dirty。這也包括使用 `@Output()` 發射事件和使用 `@HostListener` 監聽宿主元素事件。

3.  **使用 `AsyncPipe` 接收到新值：** 當元件 Template 中使用 `AsyncPipe` 訂閱 Observable 或 Promise 時，一旦 `AsyncPipe` 接收到新的值，Angular 會將該元件標記為 Dirty。

4.  **直接呼叫 `ChangeDetectorRef.markForCheck()`：** 開發者可以透過手動呼叫 `ChangeDetectorRef.markForCheck()` 方法，將元件標記為 Dirty。這種方式通常在 `OnPush` 策略下使用，用於在特定情況下手動觸發變更檢測。

5.  **`@defer` 區塊的狀態變化：** 當 Angular 17 版本引入的 `@defer` 延遲載入區塊的狀態發生變化時 (例如從 `placeholder` 狀態切換到 `complete` 狀態)，Angular 也會觸發相關元件的 Dirty 標記。

值得注意的是，當一個元件被標記為 "dirty" 時，這個標記會**向上冒泡 (Bubble Up)**。也就是說，如果一個子元件被標記為 Dirty，那麼它的父元件也會被自動標記為 Dirty，並且這個冒泡過程會一直向上傳遞到根元件。實現這個冒泡行為的內部方法是 `markViewDirty()`。

## `ChangeDetectorRef` 的角色：手動控制變更偵測的利器

在某些複雜的場景下，Angular 的自動變更偵測機制可能無法完全滿足我們的需求。這時，我們就需要借助 `ChangeDetectorRef` 這個工具，來**手動控制變更偵測的行為**。

`ChangeDetectorRef` 是一個 Angular 提供的服務，每個元件實例都擁有一個對應的 `ChangeDetectorRef` 實例。透過 `ChangeDetectorRef`，我們可以實現更精細的變更檢測控制，例如：

*   **`detach()`：**  `detach()` 方法用於**分離 (Detach)** 當前元件的變更檢測器。當一個元件的變更檢測器被分離後，Angular 將**不再自動對該元件及其子元件進行變更檢測**。即使元件被標記為 Dirty，也不會觸發畫面更新。`detach()` 方法可以用於在某些特定場景下，**暫時關閉**元件的變更檢測，以提升效能。

*   **`reattach()`：** `reattach()` 方法用於**重新附加 (Reattach)** 之前被分離的變更檢測器。當我們希望**重新啟用**元件的自動變更檢測時，可以呼叫 `reattach()` 方法。

*   **`markForCheck()`：** `markForCheck()` 方法用於將當前元件標記為 Dirty，並且會**向上遍歷所有祖先元件**，也將它們標記為 Dirty。`markForCheck()` 並不會立即觸發變更檢測，而是**等待下一個變更檢測週期**到來時，才會更新被標記的元件。`markForCheck()` 方法通常在 `OnPush` 策略下使用，用於在元件數據發生變化時，手動通知 Angular 觸發變更檢測。

*   **`detectChanges()`：** `detectChanges()` 方法用於**立即觸發**當前元件及其子元件的變更檢測。`detectChanges()` 會**同步地**執行變更檢測，並**立即更新畫面**。與 `markForCheck()` 不同，`detectChanges()` 不會向上冒泡標記 Dirty，而是**向下檢查**子元件是否為 Dirty，並進行更新。

在絕大多數情況下，Angular 的自動變更偵測機制已經足夠應付開發需求。**只有在使用了 `OnPush` 變更偵測策略，或者在需要對變更檢測進行更精細控制的特殊場景下，我們才需要使用 `ChangeDetectorRef`**。

## NgZone 與 `onMicrotaskEmpty`：揭秘通知機制的底層原理

現在我們已經了解了 Zone.js 在 Angular 變更偵測中的核心作用，但還有一個關鍵問題需要解答：**NgZone 究竟是如何通知 Angular 觸發變更檢測的呢？**

答案就藏在 NgZone 的一個自定義事件中：**`onMicrotaskEmpty`**。

Zone.js 和 NgZone 雖然密切相關，但在細節上還是存在一些區別。NgZone 在 Zone.js 的基礎上，額外提供了一些 Angular 專用的 API 和事件，`onMicrotaskEmpty` 就是其中之一。

`onMicrotaskEmpty` 事件會在 **Microtask 隊列為空** 時觸發。Microtask 隊列是 JavaScript 事件循環中的一個重要組成部分，用於處理 Promise 的 resolve/reject 回調、`MutationObserver` 的回調等微任務。當瀏覽器執行完所有當前 MacroTask (例如 DOM 事件處理、`setTimeout` 回調等) 後，就會開始處理 Microtask 隊列中的任務。只有當 Microtask 隊列為空時，瀏覽器才會進入下一個事件循環週期。

NgZone 正是利用了 `onMicrotaskEmpty` 事件，來**判斷何時觸發 Angular 的變更檢測**。當 NgZone 監聽到 `onMicrotaskEmpty` 事件時，就表示當前事件循環週期中的所有非同步任務 (包括 MacroTask 和 Microtask) 都已處理完畢，此時應用程式的狀態可能發生了變化，需要進行變更檢測，更新畫面。

讓我們再次回到程式碼，看看 `ApplicationRef` (Angular 應用程式的根服務) 是如何利用 `onMicrotaskEmpty` 事件觸發變更檢測的：

```ts
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

這段程式碼展示了 `ApplicationRef` 的構造函數。在構造函數中，`ApplicationRef` 訂閱了 `this._zone.onMicrotaskEmpty` 事件。當 `onMicrotaskEmpty` 事件觸發時，`ApplicationRef` 就會執行以下操作：

1.  呼叫 `this._zone.run(() => { ... })`，確保後續的程式碼在 NgZone 的上下文中執行。
2.  在 NgZone 的上下文中，呼叫 `this.tick()` 方法，觸發變更檢測。

## `ApplicationRef.tick()`：變更檢測的真正觸發點

現在，讓我們繼續深入程式碼，探究 `ApplicationRef.tick()` 方法的內部實現：

```ts
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

這段程式碼揭示了 `ApplicationRef.tick()` 方法的核心邏輯：

1.  `this._runningTick = true;`：設定一個標誌位，表示當前正在執行變更檢測。
2.  `for (let view of this._views) { view.detectChanges(); }`：**遍歷所有根視圖 (Root View)，並在每個根視圖上呼叫 `detectChanges()` 方法**。這裡的 `view` 實際上指的是根元件的視圖實例。換句話說，`ApplicationRef.tick()` 最終會**在根元件上執行 `detectChanges()` 方法**，從而觸發整個應用程式的變更檢測。
3.  `if (typeof ngDevMode === 'undefined' || ngDevMode) { ... }`：在開發模式下，額外執行 `checkNoChanges()` 方法，用於檢測在變更檢測過程中是否又產生了新的數據變化，以幫助開發者Debug。

因此，我們可以得出結論：**`ApplicationRef.tick()` 方法是 Angular 變更檢測的真正觸發點**。NgZone 透過監聽 `onMicrotaskEmpty` 事件，間接地呼叫 `ApplicationRef.tick()` 方法，從而驅動整個應用程式的變更檢測流程。

## Zone 的必要性反思：手動觸發與控制

透過以上的分析，我們了解到 Angular 的變更檢測機制在很大程度上依賴於 Zone.js 和 NgZone。但你或許會產生疑問：**觸發 Change Detection 一定要有 Zone 嗎？**

答案是否定的。雖然 NgZone 在 Angular 的變更檢測中扮演著重要的角色，但**Change Detection 的執行本身並不依賴 NgZone**。實際上，我們可以透過以下方式**手動觸發 Change Detection**：

1.  **使用 `ChangeDetectorRef.detectChanges()`：**  直接在元件實例上呼叫 `detectChanges()` 方法，可以立即觸發當前元件及其子元件的變更檢測。
2.  **呼叫 `ApplicationRef.tick()`：**  取得 `ApplicationRef` 服務實例，並呼叫其 `tick()` 方法，可以觸發整個應用程式的變更檢測。

這意味著，即使在沒有 Zone.js 的環境下，我們仍然可以透過手動的方式觸發 Angular 的變更檢測。事實上，Angular 團隊也在積極探索 **Zoneless Angular** 的可能性，希望在未來的版本中，能夠移除對 Zone.js 的依賴，提供更輕量級、更高效的變更檢測機制。

## Signals 的革新：精準、高效的變更偵測新範式

Angular Signals 的出現，正是 Angular 團隊在變更檢測機制上的一次重大革新。Signals 是一種全新的響應式原語 (Reactive Primitive)，它提供了一種**更精準、更高效**的變更偵測方式，並且**不再需要依賴 NgZone**。

讓我們來看一個使用 Signals 的範例：

**(程式碼範例部分，與初稿相同，請參考初稿內容)**

在這個例子中，我們使用 `signal(0)` 創建了一個 Signals 信號 `value`，並在 Template 中透過 `{{ value() }}` 的方式讀取信號的值。當點擊按鈕時，我們呼叫 `this.value.update(v => v + 1)` 來更新信號的值。

那麼，Signals 是如何改變變更檢測的運作方式的呢？讓我們借助 GIF 動畫 (CD04.gif) 來理解 Signals 理想情況下的變更流程：

![](/public/img/CD04.gif)
⬆️ (CD04.gif)

從動畫中可以看出，Signals 的變更檢測流程與傳統的 Default 策略截然不同。Signals 為變更檢測帶來了以下關鍵改變：

1.  **Consumer Dirty 標記：** 當 Template 中的 `signal()` 偵測到數值變動時，**Signal 會在內部將自身標記為 Consumer Dirty**。這個標記表示該 Signal 的值已經發生了變化，需要通知相關的元件進行更新。

2.  **HasChildViewsToRefresh 標記：** 當元件被標記為 **Consumer Dirty** 時，Angular 會呼叫 `markAncestorsForTraversal()` 方法，將其**父元件標記為 HasChildViewsToRefresh**。這種標記機制**類似於傳統 Change Detection 的 Dirty 冒泡**，但更加精細。

3.  **更聰明的 Change Detection：** 當 Change Detection 執行時，遇到被標記為 **HasChildViewsToRefresh** 的元件時，**不會深入檢查其子元件**，而是直接跳過。只有當 Change Detection 遇到被標記為 **Consumer Dirty** 的元件時，才會真正進行更新。這種機制避免了對整個元件樹的無謂檢查，極大地提升了變更檢測的效能。

4.  **擺脫 `NgZone` 的依賴：** Signals 的變更偵測機制**不再需要依賴 NgZone 來通知框架觸發檢查**，而是採用了一種**更直接、更高效**的方式。當 Signal 的值發生變化時，Signal 會主動通知 Angular 框架，觸發必要的更新流程。

## Signals 的混合情境：點擊事件與冒泡的權衡

雖然 Signals 為變更檢測帶來了革命性的改變，但在某些情境下，我們仍然需要考慮到 Angular 舊有的變更偵測規則。例如，當我們透過**點擊事件**來變更 Signals 的值時：

![](/public/img/CD05.gif)
⬆️ (CD05.gif)

你會發現，即使使用了 Signals，**「冒泡」行為依然存在**。這是因為在這種情境下，Angular **舊有的變更偵測規則仍然生效**。點擊事件仍然會觸發傳統的 Dirty 標記和冒泡機制，導致整個元件樹被檢查。

因此，在透過點擊事件等用戶互動事件觸發 Signals 變更時，Signals 的變更檢測行為與原本 `OnPush` 策略相比，**在效能上的差異並不明顯**。

**Signals 的真正優勢，在於避免非同步操作的「冒泡」**。對於以下這些**非同步操作**觸發的 Signals 變更，Signals 可以實現**更精準、更高效**的變更檢測，**不會產生「冒泡」現象**：

*   `setInterval`, `setTimeout` (定時器)
*   `Observable.subscribe`, `toSignal(Observable)` (例如 `HttpClient` 請求)
*   `RxJS fromEvent`, `Renderer2.listen` (事件流)

在這些非同步操作場景下，Signals 能夠精確地定位到數據發生變化的元件，只更新必要的元件，避免了不必要的元件樹遍歷，從而實現了顯著的效能提升。

## 沒有 Zone 的 Angular

隨著 Signals 的成熟和完善，Angular 團隊也在積極推進 **Zoneless Angular** 的實驗性特性。Zoneless Angular 的目標是**徹底移除對 Zone.js 的依賴**，完全基於 Signals 和內建的 Notify 機制來實現變更檢測。

在 Zoneless Angular 中，變更檢測的流程將會更加簡潔高效：

![](/public/img/CD06.gif)
⬆️ (CD06.gif)

在 Zoneless 模式下，**`notify()` 方法將成為觸發變更檢測的關鍵**。那麼，什麼情況下 `notify()` 會被呼叫呢？

在 Zoneless Angular 中，**`notify()` 主要在以下三個情境下被觸發**：

1.  **Template 讀取 Signals 值：** 當 Angular 編譯器檢測到 Template 中讀取了 Signals 的值 (例如 `{{ value() }}`) 時，會在編譯後的程式碼中插入對 `markAncestorsForTraversal()` 的呼叫。`markAncestorsForTraversal()` 方法最終會觸發 `notify()`，通知框架進行必要的更新。

2.  **元件被標記為 Dirty：**  當元件透過 `markViewDirty()` 函數被標記為 Dirty 時，`markViewDirty()` 內部會呼叫 `notify()`，通知框架該元件需要進行變更檢測。觸發 `markViewDirty()` 的情境包括 `@Input` 變更、Template 綁定的事件觸發、`AsyncPipe` 接收到新值、手動呼叫 `ChangeDetectorRef.markForCheck()` 和 `@defer` 區塊狀態變化等。

3.  **AfterRender Hooks 與 View 生命周期：** 當 `afterRender` hook 被註冊、View 重新附加到 Change Detection Tree 或從 DOM 移除時，Angular 也會呼叫 `notify()`。**需要注意的是，在這種情境下，`notify()` 僅僅是用於執行 hooks，而不會刷新 View**。

## 邁向 Zoneless：Standalone 應用的實踐

如果你想體驗 Zoneless Angular 的魅力，可以在 Standalone 應用程式中啟用 `provideExperimentalZonelessChangeDetection()` 特性：

```ts
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

只需要在 `ApplicationConfig` 的 `providers` 中加入 `provideExperimentalZonelessChangeDetection()`，並從 `angular.json` 中移除 `zone.js` 的 polyfills，你的 Angular 應用程式就能在 Zoneless 模式下運行了。

## Zoneless 的優劣與展望：權衡利弊，理性看待新技術

Angular Zoneless 代表了 Angular 變更檢測機制的未來發展方向。移除 Zone.js 帶來了諸多優勢：

**優點：**

*   **減少 Bundle Size：** 移除 Zone.js 可以顯著**縮減應用程式的 bundle 大小**，減少瀏覽器下載和解析 JavaScript 程式碼的時間，提升頁面載入速度。
*   **潛在的效能提升：** 移除 Zone.js 減少了額外的運行時開銷，理論上可以**提升應用程式的運行效能**。但實際的效能提升幅度需要根據具體的應用場景進行評估。
*   **保持更好的相容性：** 在某些情況下，Zone.js 可能會與一些第三方 JavaScript 函式庫產生相容性問題。移除 Zone.js 可以**提升 Angular 與其他 Library 的整合性**，並**更好地適配 Web 標準和新技術**。

**缺點與注意事項：**

*   **學習成本：** 對於已經習慣了 Zone.js 的開發者來說，適應 Zoneless 模式需要一定的學習成本。開發者需要理解 Signals 的響應式編程模型，以及 Zoneless 模式下的變更檢測觸發機制。但對於新手開發者，或者來自其他框架 (例如 React 或 Vue) 的開發者來說，Zoneless 模式可能更容易理解和上手。
*   **實驗階段：**  Angular Zoneless 目前仍處於 **實驗性階段 (experimental)**。這意味著 Zoneless 模式的 API 和行為可能會在未來的 Angular 版本中發生變化，穩定性和社群支援也可能不如傳統模式成熟。在生產環境中使用 Zoneless Angular 需要謹慎評估。
*   **遷移風險：** 將現有的 Angular 應用程式遷移到 Zoneless 模式可能需要較大的改動和測試。遷移過程可能需要重構元件的數據管理方式，並調整變更檢測策略。

**總結：** Angular Zoneless 為我們提供了一種全新的選擇。它代表著 Angular 在變更檢測機制上的持續演進和優化。在正式環境中使用 Zoneless Angular 之前，務必仔細評估其優缺點和潛在風險，並根據具體的項目需求做出合理的技術選型。

## 總結：變遷中的 Angular 變更偵測

在今天的文章中，我們一起踏上了一段探索 Angular 變更偵測機制的旅程。從最初的疑問開始，我們逐步深入到 Angular 變更檢測的各個層面，包括：

1.  **Angular 如何判斷執行變更檢測：** 我們比較了 Angular 與 Vue、React 等框架在變更偵測上的差異，理解了 Angular 的獨特之處。
2.  **Zone.js 的關鍵作用：** 我們揭秘了 Zone.js 如何攔截非同步事件，並作為 Angular 畫面更新的幕後功臣。
3.  **Default 與 OnPush 變更偵測策略：** 我們掌握了 Angular 預設的變更檢測方式，以及 OnPush 策略如何優化效能。
4.  **NgZone 的幕後運作：** 我們理解了 NgZone 如何透過 `onMicrotaskEmpty` 事件通知 Angular 進行變更檢查。
5.  **Signals 如何革新變更偵測：** 我們探索了 Signals 如何帶來更精準、高效的變更偵測，以及擺脫對 Zone.js 的依賴。
6.  **Angular Zoneless 的優缺點：** 我們評估了在 Angular 應用程式中移除 Zone.js 的優勢與潛在風險。

透過今天的學習，我們不難發現，Angular 的變更偵測機制並非一成不變，而是在不斷地演進和完善。從最初依賴 Zone.js 的 Default 策略，到透過 OnPush 策略提升效能，再到 Signals 的出現和 Zoneless Angular 的探索，Angular 團隊一直在努力為開發者提供更高效、更靈活、更易用的變更檢測機制。

目前，Angular 正處於一個重要的**過渡時期**。雖然 Zoneless Angular 已經開始進入實驗階段，但 Zone.js 仍然是 Angular 應用程式的主流選擇。在未來的一段時間內，我們很可能會看到 Zone.js 和 Signals 並存的局面。

在這個變革的過程中，開發者可能會面臨一定的衝擊和適應期。但只要我們掌握了變革的脈絡，理解了新技術的運作原理，就能更好地應對未來的挑戰，並在 Angular 的變革浪潮中乘風破浪，不斷前行。

## 附錄：延伸閱讀與深度探索

如果你想更深入地學習 Angular 變更檢測機制，或者對文章中提到的一些概念和技術感興趣，以下是一些值得閱讀的文章和資源，供你進一步探索：

1. [The Latest in Angular Change Detection](https://angular.love/the-latest-in-angular-change-detection-zoneless-signals) - 貫穿全文的主題，內容不理解的部分可以參考這裡。

2. [A change detection, zone.js, zoneless, local change detection, and signals story 📚](https://itnext.io/a-change-detection-zone-js-zoneless-local-change-detection-and-signals-story-9344079c3b9d) - 這篇更深入的探討簡報的主題，而且講的很完整。

3. [Everything you need to know about change detection in Angular](https://angular.love/everything-you-need-to-know-about-change-detection-in-angular) - 更了解 Change Detection 可以看這篇。

4. [Running change detection – Manual control](https://angular.love/running-change-detection-manual-control) - 想更手動控制 Change Detection 的可以參考這篇。

5. [Do you still think that NgZone (zone.js) is required for change detection in Angular?](https://angular.love/do-you-still-think-that-ngzone-zone-js-is-required-for-change-detection-in-angular) - 這篇解釋為什麼不需要 NgZone 也可以觸發 Change Detection。

6. [zone.js, NgZone, and ApplicationRef in Angular](https://wazeedmohammad.medium.com/zone-js-ngzone-and-applicationref-in-angular-87a9a6a95f4a) - 初入 NgZone 看這裡。

7. [I reverse-engineered Zones (zone.js) and here is what I’ve found](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found) - 深入 Zone 看這裡。

8. [Signals in Angular: deep dive for busy developers](https://angular.love/signals-in-angular-deep-dive-for-busy-developers#Signals%20as%20primitives) - 深入 Signals 看這裡。
