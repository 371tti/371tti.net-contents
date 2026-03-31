---
title: 文書検索エンジンをつくった
description: 文書検索エンジンをつくったのでその振り返り
authors:
- 371tti
tags:
- article
- search
- information-retrieval
- index
is_complete: false
---

この記事は私が2024年12月から1年くらいちょこちょこ開発してる文書解析,検索エンジンについての記事です。

# きっかけ
"作ったサイトに検索を実装したい。" とうぇぶサイトをつくったことがある人なら大体1回は思うことですが、これ結構めんどくさいですよね。  
大体解決方法としてぱっと思い浮かぶのは大体以下の3つくらいだと思います。
1. 検索用インデックスを配信してクライアント側で検索する
2. サーバー側で検索して結果を返す
3. Google Custom Search Engineなどの外部サービスを利用する

1はクライアント側で完結するので実装が簡単ですが、インデックスのサイズが大きくなったり、検索の精度を上げようとすると結構大変です。

2はサーバー側で完結するので、インデックスのサイズが大きくなってもある程度対応できますが、実装が複雑になります。

3は実装が簡単ですが、外部サービスに依存することになるので、料金的制約や、検索結果のカスタマイズが難しいなどの問題があります。

一般的に個人サイトのベストプラクティスといえば 1 ないしは 3 ですが、それは**ロマンがありません！！**

# ほしいもの
私が決めたストイックな要件は以下の通りです。
- インデックス反映はリアルタイムであること
- 検索のレスポンスは数百ms程度であること
- 検索の精度はそこそこ高いこと
- ランキングのカスタマイズが可能であること
- 複雑な検索クエリに対応すること
- 全文検索であること
- サーバーソフトに同居させること
- 日本語、英語に対応すること

これらの要件を満たす既知のソリューションでいい感じのものがなかったので、じゃあ自分でつくるか、となりました。

# できたもの(2026.3.2 現在)
現状、以下のような機能が実装できています。
- TF-IDFベクトル化を用いた全文検索エンジンの実装
- リアルタイムでインデックスを更新できるようにするためのTFとIDFの別管理
- tokenizeした単語集合を入力とすることで、言語に依存しない設計の実装
- シングルスレッドでの高速検索(2M docs でクエリによるが実用域だと思ってる)
- Booleanクエリの実装
- クエリの単語に対する重み付けの実装

[ふんわりしたベンチマーク](文書検索つくった話/benchmark.md)を取ってみました。 まぁいい感じだと思いませんか？

一部2秒とかかかっているクエリがありますがこれはストップワード(頻繁に現れる一般語 「の」「が」などの助詞とか記号) を含めていてほぼ全走になってるからですね。  
普通ストップワードは除外するか非常に高頻度のもののみを残すかするんですが、現状はストップワードの処理をしていないので、これらのクエリは全走になってしまっています。

## 改善できそうなところについて
AIに改善候補を挙げてもらいました。主に以下の5点です。
1. 位置情報（positional postings）がなく、フレーズ検索ができない。
2. クエリ実行エンジン（query planner / execution）が薄く、複雑クエリ対応が弱い。
3. 日本語検索品質を上げる解析レイヤ（正規化、同義語、表記ゆれ、ストップワードなど）が不足している。
4. ランキング拡張を支える特徴量の器（フィールド重み、静的スコアなど）が不足している。
5. マルチスレッド検索の優先度と効果を再評価する余地がある。

以下、順番に方針を書きます。
### 1. 位置情報（positional postings）について
位置情報を持てば、フレーズ検索（例: 「赤い車」）が可能になります。
ただし、全トークン数（= コーパス内の単語出現回数の総和）に比例して位置情報を保持する必要があり、インデックスサイズが大きくなります。
コストが高いため、現時点では見送る方針です。
実装する場合の案は、後の技術的な話で触れます。

### 2. クエリ実行エンジンについて
現状は、ASTベースのBooleanクエリ（`&` `|` `!` `[]`）をサポートしており、実用上は問題ありません。
これ以上の複雑なクエリに対応するには、まず1の位置情報の実装が必要だと考えています。

### 3. 日本語の検索品質を上げるための解析レイヤについて
現状は、tokenize済み単語集合を入力にすることで、言語依存処理を前段に切り出せる設計です。
そのため、tokenize段階で正規化（NFKC、大小、全半角、記号、長音、濁点など）、同義語（例: “GPU”⇔“グラボ”）、表記ゆれ、ストップワードを処理すれば、日本語検索の品質は上げられます。
ただし、ここまで内製すると複雑化しやすいため、実装コストとのバランスを見ています。

### 4. ランキングのカスタマイズについて
現状は、クエリ語への重み付けまで実装できており、実用上は問題ありません。
これ以上のカスタマイズ（フィールド別重み、文書の静的スコアなど）を支えるには、まず1の位置情報の実装が必要だと考えています。

### 5. マルチスレッドでの高速検索について
試しにマルチスレッド化しましたが、劇的な高速化は得られませんでした。
主因は、すでにメモリ帯域がボトルネックになっているためです。
そのため高速化では、並列化よりデータ表現の効率化が重要だと考えています。

また、全CPUコアを使うと他プロセスの実行余地を奪いやすく、実行バイナリ同居の用途では扱いづらくなります。
以上から、現時点ではマルチスレッド検索の優先度は高くありません。

# 技術的な話
ここからは、実装で使っている基礎技術を簡潔にまとめます。
## 基礎技術
技術スタックとスコアリングの要点です。
### 開発言語
Rustを使います。  
文書検索は大量データを扱うため、アルゴリズムだけでなく定数倍の最適化も効きます。  
低レイヤまで踏み込んだ最適化がしやすい点を重視しました。 

> [!TIP]
> Rustで開発するときはオブジェクトの関係を木構造にするように意識するとうまくいきます。
> 大体の場合これを意識すれば所有権地獄は回避できますし、パフォーマンスも出やすいです。

### TF-IDF
全文検索では、単語の重要度を TF と IDF に分けて扱います。  
まず TF（Term Frequency）は、文書内での出現の強さです。  
教科書的には次で定義されます。  

$$
TF(t, d) = \frac{f_{t,d}}{\sum_{t' \in d} f_{t',d}} 
$$

ここで $f_{t,d}$ は文書 $d$ 内の単語 $t$ の出現回数です。
> [!NOTE]
> TF の定義は実装ごとに差があります。単純な出現回数を TF として使う実装も一般的です。


IDF（Inverse Document Frequency）は、コーパス全体での希少性です。
教科書的には次で定義されます。
$$
IDF(t, D) = \log \frac{N}{|\{d \in D : t \in d\}|} 
$$
ここで $N$ は文書数、$|\{d \in D : t \in d\}|$ は語 $t$ を含む文書数です。
> [!NOTE]
> IDF も実装差が大きく、$\log$ を省略する実装もあります。

TF-IDF は TF と IDF の積で、文書内で頻出かつコーパスで希少な語ほど高くなります。

$$ TF-IDF(t, d, D) = TF(t, d) \cdot IDF(t, D) $$

TF-IDFベクトル化では、Vocabulary（単語集合）を次元とする空間に文書を写像します。  
各次元の値は対応語の TF-IDF 値で、既存のベクトル演算をそのまま使えます。

> [!IMPORTANT]
> 似たような特徴をもつ文書は似たような向きのベクトルになります。非常にあつかいやすいですね。

> [!TIP]
> TF-IDFベクトルは高次元ですが、多くの次元は 0 です。  
> 文書が語彙全体の一部しか含まないためで、疎行列最適化の余地が大きいです。

### スコアリング
TF-IDFベクトル化した文書をどう評価するかを整理します。

#### コサイン類似度
文書ベクトルの向きの近さを測る代表的な指標です。
定義は次です。
$$ 
\cos(\theta) = \frac{A \cdot B}{\|A\| \|B\|}  
$$
$$
-1 \leq \cos(\theta) \leq 1
$$

1 に近いほど向きが近く、0 は直交、-1 は逆向きです。  
正規化スコアとして扱いやすい一方、内積とノルム計算が必要で計算コストは高めです。

#### ドット積

コサイン類似度の代わりに、単純な内積を使う方法もあります。
$$ 
A \cdot B = \sum_{i=1}^{n} A_i B_i 
$$
計算は軽いですが、向きだけでなくベクトルの大きさにも強く依存します。  
そのため実運用では、文書長補正などを併用することが多いです。

#### BM25
Okapi BM25 は、文書長補正を含む実用的なスコアリング関数です。
定義は次の通りです。

$$ 
BM25(q, d) = \sum_{t \in q} IDF(t) \cdot \frac{f_{t,d} \cdot (k_1 + 1)}{f_{t,d} + k_1 \cdot (1 - b + b \cdot \frac{|d|}{avgdl})} 
$$

ここで $q$ はクエリ、$d$ は文書、$f_{t,d}$ は語 $t$ の文書内出現回数、$|d|$ は文書長、$avgdl$ は平均文書長です。  
$k_1$ は TF の効き方、$b$ は文書長補正の強さを制御します。  
今回の実装は TF と IDF を別管理しているため、BM25 も実装しやすい構成です。

## 実装
低レイヤ寄りの最適化を意識しつつ、以下を実装しています。  
- 単語集計構造体
- TFベクトル構造体
- IDFベクトル構造体
- TFベクトル, IDFベクトルをまとめた抽象構造体
- クエリ構造体
- クエリ実行エンジン実装
- ベクトル評価実装
- Serde(シリアライズ-デシリアライズ)実装

### TFベクトル構造体
`TFVector` は文書ごとの TF を保持する疎ベクトルです。  
文書数に比例して増えるため、メモリ効率を最優先にしています。  
以下が実装です。  
```rust
#[derive(Debug)]
#[repr(align(32))] // どうなんだろうか
pub struct TFVector<N> 
where N: Num
{
    inds: NonNull<u32>,
    vals: NonNull<N>,
    cap: u32,
    nnz: u32,
    len: u32,
    /// sum of terms of this document
    /// denormalize number for this document
    /// for reverse calculation to get term counts from tf values
    term_sum: u32, // for future use
}
```
[疎ベクトル](文書検索つくった話/SparseVector.md)を前提に、構造体サイズを小さく保つ設計です。  
`Vec` を2本そのまま持つと 64bit 環境で 48byte になりますが、長さと容量を `u32` で共通管理し、32byteに抑えています。

主なフィールドは次の通りです。
- `inds`: 非ゼロ要素の次元インデックス配列
- `vals`: 非ゼロ要素の TF 値配列
- `cap`: `inds` / `vals` の共通容量
- `nnz`: 非ゼロ要素数
- `len`: ベクトル次元数
- `term_sum`: 文書内総語数（逆変換用）

メソッドは通常の疎ベクトル操作が中心なので割愛します。

### IDFベクトル構造体
`IDFVector` はコーパス全体の IDF を持つ構造体です。  
文書集合が変わると再計算が必要で、全語彙分を保持するため密ベクトルになります。  
実装は以下です。  
```rust
#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct IDFVector
{
    /// IDF Vector it is not sparse because it is mostly filled
    pub idf_vec: Vec<f32>,
    /// latest entropy
    pub latest_entropy: u64,
    /// document count
    pub doc_num: u64,
}
```
主なフィールドは次の通りです。
- `idf_vec`: 全語彙の IDF 値
- `latest_entropy`: キャッシュ世代識別子
- `doc_num`: 文書数

内部が `Vec` ベースなので、API は最小限にしています。

### TFベクトル, IDFベクトルをまとめた抽象構造体
`TFIDFVectorizer` は TF・IDF と検索に必要な索引をまとめたトップレベル構造体です。
```rust
/// TF-IDF Vectorizer
///
/// The top-level struct of this crate, providing the main TF-IDF vectorizer features.
///
/// It converts a document collection into TF-IDF vectors and supports similarity
/// computation and search functionality.
///
/// ### Internals
/// - Corpus vocabulary
/// - Sparse TF vectors per document
/// - term index mapping
/// - Cached IDF vector
/// - Pluggable TF-IDF engine
/// - Inverted document index
///
/// ### Type Parameters
/// - `N`: Vector parameter type (e.g., `f32`, `f64`, `u16`)
/// - `K`: Document key type (e.g., `String`, `usize`)
/// - `E`: TF-IDF calculation engine
///
/// ### Notes
/// - Requires an `Arc<Corpus>` on construction
/// - `Corpus` can be shared across multiple vectorizers
///
/// ### Serialization
/// Supported.  
/// Serialized data includes the `Corpus` reference.
///
/// For corpus-independent storage, use [`TFIDFData`].
#[derive(Debug, Clone)]
pub struct TFIDFVectorizer<N = f16, K = String, E = DefaultTFIDFEngine>
where
    N: Num + Copy + Into<f64> + Send + Sync,
    E: TFIDFEngine<N> + Send + Sync,
    K: Clone + Send + Sync + Eq + std::hash::Hash,
{
    /// Document's TF Vector
    pub documents: IndexMap<K, TFVector<N>>,
    /// TF Vector's term dimension sample and reverse index
    /// Key is never changed and unused terms are not removed
    pub term_dim_rev_index: IndexMap<Box<str>, Vec<u32>>,
    /// Corpus reference
    pub corpus_ref: Arc<Corpus>,
    /// IDF Vector
    pub idf_cache: IDFVector,
    _marker: std::marker::PhantomData<E>,
}
```

主なフィールドは次の通りです。
- `documents`: 文書ごとの TF ベクトル
- `term_dim_rev_index`: 語と次元の対応表
- `corpus_ref`: コーパス統計への参照
- `idf_cache`: IDF のキャッシュ
- `_marker`: エンジン型保持用の `PhantomData`

`Corpus` は IDF 計算に必要な全体統計を持ちます。
```rust
/// Corpus for TF-IDF Vectorizer
///
/// Manages global document-frequency statistics required for IDF calculation.
///
/// This struct does **not** store document text or identifiers.
/// It only tracks:
/// - Total number of documents
/// - Number of documents containing each term
///
/// ### Thread Safety
/// - Fully thread-safe
/// - Implemented using `DashMap` and atomics
///
/// ### Notes
/// - Must be shared via `Arc<Corpus>`
/// - Can be reused across multiple vectorizers
#[derive(Debug, Serialize, Deserialize, Default)]
pub struct Corpus {
    /// corpus add_num
    /// for update notify
    pub add_num: AtomicU64,
    /// corpus sub_num
    /// for update notify
    pub sub_num: AtomicU64,
    // term counts in corpus
    pub term_counts: DashMap<Box<str>, u64, RandomState>,
}
```
役割はシンプルで、`add_num` / `sub_num` で更新世代を追跡し、`term_counts` で語の文書頻度を保持します。  
これを `idf_cache.latest_entropy` と比較して、IDF キャッシュの有効性を判定します。

また `TFIDFEngine` トレイトで計算部を抽象化し、実装差し替えを可能にしています。以下がデフォルト実装です。
```rust
/// TF-IDF Calculation Engine Trait
///
/// Defines the behavior of a TF-IDF calculation engine.
///
/// Custom engines can be implemented and plugged into
/// [`TFIDFVectorizer`].
///
/// A default implementation, [`DefaultTFIDFEngine`], is provided.
///
/// ### Supported Numeric Types
/// - `f16`
/// - `f32`
/// - `u16`
/// - `u32`
pub trait TFIDFEngine<N>: Send + Sync
where
    N: Num + Copy
{
    /// Method to generate the IDF vector
    /// # Arguments
    /// * `corpus` - The corpus
    /// * `term_dim_sample` - term dimension sample
    /// # Returns
    /// * `Vec<N>` - The IDF vector
    /// * `denormalize_num` - Value for denormalization
    fn idf_vec(corpus: &Corpus, term_dim_sample: &Vec<Box<str>>) -> Vec<f32> {
        let mut idf_vec = Vec::with_capacity(term_dim_sample.len());
        let doc_num = corpus.get_doc_num() as f64;
        for term in term_dim_sample.iter() {
            let doc_freq = corpus.get_term_count(term);
            idf_vec.push((doc_num / (doc_freq as f64 + 1.0)) as f32);
        }
        idf_vec
    }
    /// Method to generate the TF vector
    /// # Arguments
    /// * `freq` - term frequency
    /// * `term_dim_sample` - term dimension sample
    /// # Returns
    /// * `(ZeroSpVec<N>, f64)` - TF vector and value for denormalization
    fn tf_vec(freq: &TermFrequency, term_dim_sample: &IndexSet<Box<str>>) -> TFVector<N>;

    fn tf_denorm(val: N) -> u32;
}

/// デフォルトのTF-IDFエンジン
#[derive(Debug, Clone)]
pub struct DefaultTFIDFEngine;
impl DefaultTFIDFEngine {
    pub fn new() -> Self {
        DefaultTFIDFEngine
    }
}

impl TFIDFEngine<f16> for DefaultTFIDFEngine {
    #[inline]
    fn tf_vec(freq: &TermFrequency, term_dim_sample: &IndexSet<Box<str>>) -> TFVector<f16> {
        // Build sparse TF vector: only non-zero entries are stored
        let term_sum = freq.term_sum() as u32;
        let len = freq.term_num();
        let mut ind_vec: Vec<u32> = Vec::with_capacity(len);
        let mut val_vec: Vec<f16> = Vec::with_capacity(len);
        for (term, count) in freq.iter() {
            let count = (count as f32).sqrt();
            if let Some(idx) = term_dim_sample.get_index(term) {
                ind_vec.push(idx as u32);
                val_vec.push(f16::from_f32(count));
            }
        }
        unsafe { TFVector::from_vec(ind_vec, val_vec, len as u32, term_sum) }
    }

    #[inline(always)]
    fn tf_denorm(val: f16) -> u32 {
        val.to_f32().pow(2) as u32
    }
}

impl TFIDFEngine<f32> for DefaultTFIDFEngine
{
    #[inline]
    fn tf_vec(freq: &TermFrequency, term_dim_sample: &IndexSet<Box<str>>) -> TFVector<f32> {
        // Build sparse TF vector: only non-zero entries are stored
        let term_sum = freq.term_sum() as u32;
        let len = freq.term_num();
        let mut ind_vec: Vec<u32> = Vec::with_capacity(len);
        let mut val_vec: Vec<f32> = Vec::with_capacity(len);
        for (term, count) in freq.iter() {
            if let Some(idx) = term_dim_sample.get_index(term) {
                ind_vec.push(idx as u32);
                val_vec.push(count as f32);
            }
        }
        unsafe { TFVector::from_vec(ind_vec, val_vec, len as u32, term_sum) }
    }

    #[inline(always)]
    fn tf_denorm(val: f32) -> u32 {
        val as u32
    }
}

impl TFIDFEngine<u32> for DefaultTFIDFEngine
{
    #[inline]
    fn tf_vec(freq: &TermFrequency, term_dim_sample: &IndexSet<Box<str>>) -> TFVector<u32> {
        // Build sparse TF vector: only non-zero entries are stored
        let term_sum = freq.term_sum() as u32;
        let len = freq.term_num();
        let mut ind_vec: Vec<u32> = Vec::with_capacity(len);
        let mut val_vec: Vec<u32> = Vec::with_capacity(len);
        for (term, count) in freq.iter() {
            if let Some(idx) = term_dim_sample.get_index(term) {
                ind_vec.push(idx as u32);
                val_vec.push(count as u32);
            }
        }
        unsafe { TFVector::from_vec(ind_vec, val_vec, len as u32, term_sum) }
    }

    #[inline(always)]
    fn tf_denorm(val: u32) -> u32 {
        val
    }
}

impl TFIDFEngine<u16> for DefaultTFIDFEngine
{
    #[inline]
    fn tf_vec(freq: &TermFrequency, term_dim_sample: &IndexSet<Box<str>>) -> TFVector<u16> {
        // Build sparse TF vector: only non-zero entries are stored
        let term_sum = freq.term_sum() as u32;
        let len = freq.term_num();
        let mut ind_vec: Vec<u32> = Vec::with_capacity(len);
        let mut val_vec: Vec<u16> = Vec::with_capacity(len);
        for (term, count) in freq.iter() {
            if let Some(idx) = term_dim_sample.get_index(term) {
                ind_vec.push(idx as u32);
                val_vec.push(count as u16);
            }
        }
        unsafe { TFVector::from_vec(ind_vec, val_vec, len as u32, term_sum) }
    }

    #[inline(always)]
    fn tf_denorm(val: u16) -> u32 {
        val as u32
    }
}
```

複数の数値型に対応するため実装は型ごとに分かれていますが、処理フロー自体は共通です。  
IDF は全実装で同じ計算を使い、差分は TF の保持形式です。

- `f16`: `sqrt(count)` を保持し、高頻度語を強く圧縮
- `f32`: `count` をそのまま保持
- `u32`: `count` を整数で保持
- `u16`: `count` を `u16` で保持（値域上限に注意）

この設計により、メモリ使用量と精度のトレードオフを型で選べます。  
`f16` で `log` ではなく平方根を使うのは演算コストを抑えるためです。大量語処理ではこちらが有利です。

明日もかく