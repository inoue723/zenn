---
title: "dayjsのtzのバグを調べてみた"
emoji: "🐕"
type: "tech"
topics: ['dayjs', 'javascript']
published: false
publication_name: "backtech"
---
# 気になる挙動
tzとstartOfを組み合わせると、特定のTZとシステムロケールの組み合わせで、想定外の動きをすることに気づきました。
以下、システムロケールはすべて日本時間とします。

コード
```typescript
import dayjs from 'dayjs';
import timezone from 'dayjs/plugin/timezone.js';
import utc from 'dayjs/plugin/utc.js';

dayjs.extend(utc);
dayjs.extend(timezone);

const now = dayjs('2024-02-11 07:00:00+09');
const abidjanNow = now.tz('Africa/Abidjan').format();
const abidjanStartOfDay = now.tz('Africa/Abidjan').startOf('day').format();

console.table({ abidjanNow, abidjanStartOfDay });
```

結果

```
┌───────────────────┬─────────────────────────────┐
│ (index)           │ Values                      │
├───────────────────┼─────────────────────────────┤
│ abidjanNow        │ '2024-02-10T22:00:00Z'      │
│ abidjanStartOfDay │ '2024-02-09T15:00:00Z'      │
└───────────────────┴─────────────────────────────┘
```
abidjanStartOfDayが予想では`2024-02-10T00:00:00Z`になると思ったのですが、異なります。
ちなみにmomentjsだと予想どおり`2024-02-10T00:00:00Z`になります。

# 原因を探る
githubに関連するissueが何個か挙がっていました。いずれも解決してないようです。
休日に暇だったので、原因を探ることにしました。

ソースコードでstartOfの実装を見てみます。
リンクはこちらです。(執筆時点での最新のv1.11.0です)
https://github.com/iamkun/dayjs/blob/e56a8703dcc3820196972f53990010daf248726b/src/plugin/timezone/index.js#L118-L126

この中で怪しいのは、125行目の`startOfWithoutTz.tz(this.$x.$timezone, true)`ですね！
(実は地道にログを仕込んだり、node inspect使って2hくらいデバッグしていました。。。)

tz()の第２引数はkeepLocalTimeです。これは例えば
```typescript
console.log(dayjs().format());
console.log(dayjs().tz('Asia/Ho_Chi_Minh', true).format());
// 2024-02-11T07:00:00+09:00
// 2024-02-11T07:00:00+07:00 <--
```
こんな感じでタイムゾーン部分だけを変えたいケースで使います。

試しに問題のtzで変換をしてみます。
```typescript
console.log(dayjs('2024-02-11 07:00:00+09').tz('Africa/Abidjan', true).format());
// 2024-02-10T22:00:00Z
```

`2024-02-11T07:00:00Z`にならないです。まずいですね。
ただ実はこれ、toDateの結果は正しいんです。
```typescript
console.log(dayjs('2024-02-11 07:00:00+09').tz('Africa/Abidjan', true).toDate().toISOString())
// 2024-02-11T07:00:00.000Z
```
つまりformatに何かしら問題ありそうという感じですね。

なんだformatの挙動がおかしいだけか、と思ったら、
```typescript
const abidjan = dayjs().tz('Africa/Abidjan');
console.log(abidjan.startOf('day').toDate().toISOString())
console.log(abidjan.startOf('day').startOf('day').toDate().toISOString())
console.log(abidjan.startOf('day').startOf('day').startOf('day').toDate().toISOString())
// 2024-02-10T00:00:00.000Z
// 2024-02-09T00:00:00.000Z
// 2024-02-08T00:00:00.000Z
```
startOfしてるだけなのにどんどん過去になる・・・。
これは、先程のstartOfのソースコードの
https://github.com/iamkun/dayjs/blob/e56a8703dcc3820196972f53990010daf248726b/src/plugin/timezone/index.js#L118-L126
L123でformatを使ってるからですね。

# 既に解決するPRが投げられていた
ここまで調べて、issueを再度漁っていると、
https://github.com/iamkun/dayjs/pull/2264

既に修正するPRが作られていました。
どうやらutcoffsetが0のタイムゾーン(Africa/Abidjanなど)のdayjsオブジェクトに対して、$u(utcフラグ)がtrueになっていることで、format時に2重でoffsetが引かれたような動作になってるみたいです。

早くマージしてほしいですね。

:::details tz処理後のDateオブジェクトについて余談
本来tzで処理したdayjsオブジェクトは、以下のような値をもっています。
```typescript
dayjs('2024-02-11 00:00:00Z').tz("Africa/Abidjan")
/*
M {
  '$L': 'en',
  '$d': 2024-02-10T15:00:00.000Z,
  '$y': 2024,
  '$M': 1,
  '$D': 11,
  '$W': 0,
  '$H': 0,
  '$m': 0,
  '$s': 0,
  '$ms': 0,
  '$x': { '$timezone': 'Africa/Abidjan' },
  '$isDayjsObject': true,
  '$offset': 0,
  '$u': true
}
*/
```
$dにdateオブジェクトを格納しているのですが、9時間ズレてるように見えますね。
しかしこれは問題ないです。実際にformatする際には`$D`(日にち), `$H`(時間)などが使われ、`$d`の値は参照されないです。
また、valueOfでunix時間を出力するときには、システムロケールとの差分が足されることで、正しい値になります。

formatをシンプルに保つためにこんな実装なのかなと思いますが、このせいでだいぶ混乱しました。。。
<!-- https://github.com/iamkun/dayjs/blob/e56a8703dcc3820196972f53990010daf248726b/src/plugin/utc/index.js#L115-L119 -->
<!-- `this.$d.getTimezoneOffset()`が例えば上の例だと-540(9時間)になる。 -->
:::

:::details format時に2重でoffsetが引かれるとは？
format時に2重でoffsetが引かれるというのは正確に言い直すと、tz処理後にcloneした時点で、isUtcがtrueの場合は、`$H`などに`$d.getUTCHours()`でutcに変換された値が代入されます。
https://github.com/iamkun/dayjs/blob/e56a8703dcc3820196972f53990010daf248726b/src/plugin/utc/index.js#L55-L69
そうすると、本来は$d.getHours()でシステムロケールとの差分を考慮して代入されるところが、utcになってしまい、システムロケールとutcの差分の分がズレてしまいます。

つまり上の例を更にcloneすると、
```typescript
dayjs('2024-02-11 00:00:00Z').tz("Africa/Abidjan").clone()
/*
M {
  '$L': 'en',
  '$d': 2024-02-10T15:00:00.000Z,
  '$y': 2024,
  '$M': 1,
  '$D': 10,
  '$W': 6,
  '$H': 15, $d.getUTCHours()で取得した値。これがformatに使われる。
  '$m': 0,
  '$s': 0,
  '$ms': 0,
  '$x': { '$timezone': 'Africa/Abidjan' },
  '$isDayjsObject': true,
  '$offset': 0,
  '$u': true
}
*/
```
```typescript
dayjs('2024-02-11 00:00:00Z').tz("Africa/Abidjan").clone().format()
// 2024-02-10T15:00:00Z
```
:::