## はじめに
vueでコンポーネント設計、してますか？
変更容易性のあるコードを書くためには、良いコンポーネントに分割されていることが必要です。
今回は進捗度を表示するコンポーネントを例にコンポーネントの分割、テストの書き方を整理します。


## 進捗度を表示するコンポーネントを作る
それでは実際にコードを書いていきます。
propsで受け取った到着見込み時間と現在時刻の違いに応じて、メッセージを表示させるコンポーネントを作ります。

### 使用技術スペック
vue　v3.3, vitest v0.29.2, vue-testing-library v6.6.1, Typescript

```vue
<script setup lang="ts">
import { computed, ComputedRef, Ref, toRef } from "vue";
const props = defineProps<{
  eta: Date
}>()

const now = new Date();
const remainingTimeInSeconds = computed(()=> etaRef.value.getTime() - now.getTime());

const progress = computed<number>(() => {
    if (remainingTimeInSeconds < 0) {
      return 0;
    } else if (remainingTimeInSeconds < 60 * 1000 * 10) {
      return Math.floor(remainingTimeInSeconds / 1000 / 60);
    } else {
      return 10;
    }
  });
  return progress;
)
</script>

<template>
    <h1 v-if="progress > 9">まだまだ先🍤</h1>
    <h1 v-else-if="progress > 0 && progress <= 9">もうすぐつくよ🧋</h1>
    <h1 v-else="progress == 0">ついたよ🍜</h1>
</template>
```
## 現状のデメリット
この程度の量であればあまり問題にならないように思われますが、テストを書く観点から考えると、UIとビジネスロジックを分けた方がテストが書きやすくなります。
例えば以下のようなテストを書いた場合、
```vue
import { describe, it, vi, beforeEach, afterEach } from "vitest";
import { computed } from "vue";
import { render, screen } from "@testing-library/vue";
import ProgressBar from "./ProgressBar.vue";
import { calculateRemainingTime } from "./calculateRemaining";

describe("残り時間に合わせて状況が正しく表示される", () => {
  beforeEach(() => {
    // vitestで時間を固定する
    vi.useFakeTimers();
  });

  afterEach(() => {
    // 時間の固定を解除
    vi.useRealTimers();
  });

  it("残り時間10分のとき、まだまだと表示される", async () => {
    render(ProgressBar, {
      props: {
        //2023年9月30日12時を予想時刻としてセット
        eta: new Date(2023, 9, 30, 12, 0),
      },
    })

    const date = new Date(2023, 9, 30, 11, 50)
    vi.setSystemTime(date)

    screen.getByText('まだまだ先🍤')
  });
});
```
このコードが失敗したときに、
* ビジネス層（現在時刻と到着見込み時間の時間差を出す計算）でエラーが起きたのか？
* 計算結果が間違っていて、UI層で表示にバグが生じたのか？
* ビジネスロジックとUIの両方に修正しなければならないバグが生じているのか？
以上のうち、どれがテスト失敗の原因となったのかを調査しなければなりません。
これではテストの粒度が十分高いとは言えず、修正に時間がかかってしまいます。

加えて、UI層とビジネスロジックが結合していると、以下のようなデメリットも考えられます。
* どちらも再利用が難しくなる
* ビジネスロジック・UIのどちらかを変更する際に、片方の変更が変更しなくて良いはずの部分に影響を与えてしまう恐れが強くなる

それではUIとビジネスロジックを分離したコードを書いてみましょう。

# UIとビジネスロジックの分離
```vue
//calculateRemaining.ts
import { computed, Ref, toRef } from "vue";

export const useCalculateRemainingTime = (eta: Date) => {
  const etaRef: Ref<Date> = toRef(eta);
  const now = new Date();
  const remainingTimeInSeconds: number = etaRef.value.getTime() - now.getTime();
  const progress = computed<number>(() => {
    if (remainingTimeInSeconds < 0) {
      return 0;
    } else if (remainingTimeInSeconds < 60 * 1000 * 10) {
      return Math.floor(remainingTimeInSeconds / 1000 / 60);
    } else {
      return 10;
    }
  });
  return progress;
};

//Progress.vue
<script setup lang="ts">
import { useCalculateRemainingTime } from "./calculateRemaining"
import { computed, ComputedRef, Ref, toRef } from "vue";
const props = defineProps<{
  eta: Date
}>()

const progress:ComputedRef<number> = useCalculateRemainingTime(props.eta)

</script>

<template>
    <h1 v-if="progress > 9">まだまだ先🍤</h1>
    <h1 v-else-if="progress > 0 && progress <= 9">もうすぐつくよ🧋</h1>
    <h1 v-else="progress == 0">ついたよ🍜</h1>
</template>

//テストコード
import { expect, describe, it, vi, beforeEach, afterEach } from "vitest";
import { computed } from "vue";
import { render, screen } from "@testing-library/vue";
import ProgressBar from "./ProgressBar.vue";
import { calculateRemainingTime } from "./calculateRemaining";

describe("残り時間に合わせて状況が正しく表示される", () => {
  beforeEach(() => {
    // vitestで時間を固定する
    vi.useFakeTimers();
  });

  afterEach(() => {
    // 時間の固定を解除
    vi.useRealTimers();
  });

  it("残り時間10分のとき、まだまだと表示される", async () => {
    render(ProgressBar, {
      props: {
        eta: new Date(2023, 9, 30, 12, 0),
      },
    })

    const mock = vi.fn().mockImplementation(calculateRemainingTime)
    mock.mockImplementationOnce(() => computed(()=>10))

    screen.getByText('まだまだ先🍤')
  });

  it ("現在時刻と渡された時刻の差を計算する", ()=>{
    //現在時刻をセット
    const date = new Date(2023, 9, 30, 11, 50)
    vi.setSystemTime(date)

    const eta = new Date(2023, 9, 30, 12, 0)
    const progress = calculateRemainingTime(eta)
    expect(progress).toBe(1000 * 60 * 10)
  })
});
```

## ビジネス層をCompositionAPIとして分離したことによる改善点 
* テストの失敗が何によるものか不明 -> **テストを分離できたので、明確化**
* どちらも再利用が難しくなる -> ***再利用しやすくなった！**
* ビジネスロジック・UIのどちらかを変更する際に、片方の変更が変更しなくて良いはずの部分に影響を与えてしまう恐れが強くなる -> **分離されたことにより、その恐れはない**

## まとめ
以上のように小さく、単一責任の原則を適用しながらコードを書くことで、保守性の高いコードを書くことができます。
加えて最初からテストの書きやすいコードを意識していくことで、テストしやすく変更、リファクタリングしやすくなります。

## 参考
https://ics.media/entry/210929/
https://lachlanmiller.gumroad.com/l/vuejs-composition-api
