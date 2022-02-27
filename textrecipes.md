---
title: "Rによる自然言語処理（RcppMeCab, neologd, textrecipes, XGBoost）"
author: "paithiov909"
date: "2022-02-27"
output: html_document
---



## この記事について

[以前に書いた記事](https://github.com/paithiov909/wabbitspunch/blob/master/content/articles/about.md)を焼き直ししつつ、ばんくしさんの以下のブログ記事のまねをRでやってみます。

- [Rustによるlindera、neologd、fasttext、XGBoostを用いたテキスト分類 - Stimulator](https://vaaaaaanquish.hatenablog.com/entry/2020/12/14/192246)

ばんくしさんの記事は「Pythonどこまで脱却できるのか見るのも兼ねて」ということで、Rustで自然言語処理を試しています。私はべつに自然言語処理を実務でやるエンジニアとかではないですが、PythonじゃなくてRustとかGoといった静的型付けで速い言語で安全に書けたらうれしい場面があるよね、みたいなモチベーションなのかなと想像しています。

実際のところ、自分でコードを書きながら自然言語処理の真似事をするなら依然としてPythonが便利です。Rと比べても、Pythonには[SudachiPy](https://github.com/WorksApplications/SudachiPy)や[janome](https://mocobeta.github.io/janome/)といった選択肢がある一方で、RにはRコンソールからのみで導入が完了する形態素解析の手段が（少なくともCRANには）ありません。自然言語処理をやる言語としてPythonのほうがメジャーなことにはほかにもいくつかの理由というか経緯があるのでしょうが、Pythonを採用したほうがよいひとつのモチベーションとしては、テキストマイニングして得た特徴量を投入してディープラーニングをしたい場合は事実上Pythonを選択するしかないというのもある気がします。一応、[{keras}](https://keras.rstudio.com/)や[{torch}](https://github.com/mlverse/torch)というのもありますが、このあたりのパッケージを使うのはまだ趣味の領域な気がします。

そうはいっても、[{RMeCab}](https://sites.google.com/site/rmecab/)は強力なツールです。なぜかRからテキストマイニングに入ってしまった人間にとって、比較的簡単に導入できてほとんど環境を問わずすぐ使えるRMeCabは欠かせないツールだったことでしょう。ただ、Rに慣れてきていろいろなことをやってみたくなると、RMeCabは「なんか使いにくいな」みたいになりがちです。Rでも自然言語処理をやるためのパッケージは以下に紹介されているようにたくさんあるのですが、そもそも日本語情報があまりないし、そこそこがんばらないと詰まります。

https://twitter.com/dataandme/status/1092509662384189441

でもまあRでもできなくはないんだよというのをしめす目的で、ここでは先の記事と同じようなことをRでやっていきます。

## パッケージの選定にあたって

この記事では、RcppMeCabをforkしたパッケージを使用して、テキストの分かち書きをしています。

形態素解析などをやるパッケージとしては他にも次に挙げるようなものがあります。知るかぎりではぜんぶ個人開発で、環境や使用する辞書、解析する文字列などによって上手く動いたり動かなかったりします。Neologd辞書を使うならRcppMeCabにすべきですが、メンテナの人が最近忙しいとかで、あまりアクティブに開発されていません。

- [IshidaMotohiro/RMeCab: Interface to MeCab](https://github.com/IshidaMotohiro/RMeCab)
- [junhewk/RcppMeCab: RcppMeCab: Rcpp Interface of CJK Morpheme Analyzer MeCab](https://github.com/junhewk/RcppMeCab)
- [uribo/sudachir: R Interface to 'Sudachi'](https://github.com/uribo/sudachir)

Universal Dependenciesなら次が使えます。udpipeはC++実装のラッパー、spacyrはPythonバックエンドです。

- [bnosac/udpipe: R package for Tokenization, Parts of Speech Tagging, Lemmatization and Dependency Parsing Based on the UDPipe Natural Language Processing Toolkit](https://github.com/bnosac/udpipe)
- [quanteda/spacyr: R wrapper to spaCy NLP](https://github.com/quanteda/spacyr)

最近になってbnosacからBPE（Byte Pair Encoding）とsentencepieceのRラッパーがリリースされました。

- [bnosac/tokenizers.bpe: R package for Byte Pair Encoding based on YouTokenToMe](https://github.com/bnosac/tokenizers.bpe)
- [bnosac/sentencepiece: R package for Byte Pair Encoding / Unigram modelling based on Sentencepiece](https://github.com/bnosac/sentencepiece)

## セットアップ


```r
require(tidymodels)
options(mecabSysDic = file.path(system("mecab-config --dicdir", intern = TRUE), "mecab-ipadic-neologd"))

tidymodels::tidymodels_prefer()
```

## データの準備

[livedoorニュースコーパス](https://www.rondhuit.com/download.html#ldcc)を使います。以下の9カテゴリです。

- トピックニュース
- Sports Watch
- ITライフハック
- 家電チャンネル
- MOVIE ENTER
- 独女通信
- エスマックス
- livedoor HOMME
- Peachy

[パーサを書いた](https://github.com/paithiov909/ldccr)ので、それでデータフレームにします。


```r
corpus <- ldccr::read_ldnws()
#> Done.
```


```r
corpus <- corpus %>%
  dplyr::select(-file_path) %>%
  dplyr::mutate(category = as.factor(category)) %>%
  dplyr::mutate(body = audubon::strj_normalize(body)) %>%
  tibble::rowid_to_column()
```

表層形の分かち書きにすることが目的なので、ここでは脳死でNEologd辞書を使います。

この記事を書いた当初はRcppMeCab + NEologdでの形態素解析を試そうとして上手くいかず、[rjavacmecab](https://github.com/paithiov909/rjavacmecab)という自作パッケージで代用していました（動くけど非常に遅い）。その後、RcppMeCabを使う場合には次のような感じでできるのを確認しました。ただし、2021年1月現在CRANにあるバージョン（0.0.1.2）は未知語の処理にバグがあるようなので、ここでは[ソースを修正したもの](https://github.com/paithiov909/RcppMeCab)を使っています。


```r
corpus <- corpus %>%
  dplyr::pull(body) %>%
  RcppMeCab::posParallel(format = "data.frame") %>%
  tidyr::drop_na() %>%
  audubon::pack() %>%
  dplyr::mutate(doc_id = as.integer(doc_id)) %>%
  dplyr::left_join(corpus, by = c("doc_id" = "rowid"))
```

こういうデータになります。


```r
corpus <- corpus %>%
  dplyr::select(doc_id, category, text) %>%
  dplyr::glimpse()
#> Rows: 7,376
#> Columns: 3
#> $ doc_id   [3m[38;5;246m<int>[39m[23m 1, 10, 100, 1000, 1001, 1002, 10…
#> $ category [3m[38;5;246m<fct>[39m[23m dokujo-tsushin, dokujo-tsushin, …
#> $ text     [3m[38;5;246m<chr>[39m[23m "友人 代表 の スピーチ 、 独 女 …
```

## モデルの学習（FeatureHashing）

データを分割します。


```r
corpus_split <- rsample::initial_split(corpus, prop = .8)
corpus_train <- rsample::training(corpus_split)
corpus_test <- rsample::testing(corpus_split)
```

以下のレシピとモデルで学習します。ここでは、ハッシュトリックを使っています。デフォルトだとパラメータは[ここに書いている感じ](https://parsnip.tidymodels.org/reference/boost_tree.html)になります。

なお、tidymodelsの枠組みの外であらかじめ分かち書きを済ませましたが、`textrecipes::step_tokenize`の`custom_token`引数に独自にトークナイザを指定することで、一つのstepとして分かち書きすることもできます。


```r
corpus_spec <-
  parsnip::boost_tree(
    sample_size = tune::tune(),
    loss_reduction = tune::tune(),
    tree_depth = tune::tune()
  ) %>%
  parsnip::set_engine("xgboost") %>%
  parsnip::set_mode("classification")

space_tokenizer <- function(x) {
  strsplit(x, " +")
}

corpus_rec <-
  recipes::recipe(
    category ~ text,
    data = corpus_train
  ) %>%
  textrecipes::step_tokenize(text, custom_token = space_tokenizer) %>%
  textrecipes::step_tokenfilter(text, min_times = 10L, max_tokens = 200L) %>%
  textrecipes::step_texthash(text, num_terms = 200L)
```


```r
corpus_wflow <-
  workflows::workflow() %>%
  workflows::add_model(corpus_spec) %>%
  workflows::add_recipe(corpus_rec)
```

精度（accuracy）をメトリクスにして学習します。5分割CVで、簡単にですが、ハイパーパラメータ探索をします。


```r
doParallel::registerDoParallel(cores = parallel::detectCores() - 1)

corpus_tune_res <-
  corpus_wflow %>%
  tune::tune_grid(
    resamples = rsample::vfold_cv(corpus_train, v = 5L),
    grid = dials::grid_latin_hypercube(
      dials::sample_prop(),
      dials::loss_reduction(),
      dials::tree_depth(),
      size = 5L
    ),
    metrics = yardstick::metric_set(yardstick::accuracy),
    control = tune::control_grid(save_pred = TRUE)
  )

doParallel::stopImplicitCluster()
```

ハイパラ探索の要約を確認します。


```r
ggplot2::autoplot(corpus_tune_res)
```

![plot of chunk autoplot](figure/autoplot-1.png)

`fit`します。


```r
corpus_wflow <-
  tune::finalize_workflow(corpus_wflow, tune::select_best(corpus_tune_res, metric = "accuracy"))

corpus_fit <- parsnip::fit(corpus_wflow, corpus_train)
#> [05:44:58] WARNING: amalgamation/../src/learner.cc:1115: Starting in XGBoost 1.3.0, the default evaluation metric used with the objective 'multi:softprob' was changed from 'merror' to 'mlogloss'. Explicitly set eval_metric if you'd like to restore the old behavior.
```

学習したモデルの精度を見てみます。


```r
dplyr::select(corpus_test, category) %>%
  dplyr::bind_cols(predict(corpus_fit, corpus_test)) %>%
  yardstick::accuracy(truth = category, estimate = .pred_class)
#> # A tibble: 1 × 3
#>   .metric  .estimator .estimate
#>   <chr>    <chr>          <dbl>
#> 1 accuracy multiclass     0.854
```

## 所感

このコーパスのカテゴリ分類はかなり易しいタスクであることが知られている（というか、一部のカテゴリではそのカテゴリを同定できる単語が本文に含まれてしまっている）ので相性もあるのでしょうが、ハッシュトリックしてXGBoostに投入するだけで簡単によい精度の予測ができる点は気持ちよいです。

一方で、RcppMeCabは使えるようにするまでが依然としてめんどくさいということもあり、分かち書きするだけならほかの手段のほうが簡単な気がします。

## セッション情報


```r
sessioninfo::session_info()
#> ─ Session info ──────────────────────────────────
#>  setting  value
#>  version  R version 4.1.2 (2021-11-01)
#>  os       Ubuntu 18.04.5 LTS
#>  system   x86_64, linux-gnu
#>  ui       X11
#>  language (EN)
#>  collate  en_US.UTF-8
#>  ctype    en_US.UTF-8
#>  tz       Etc/UTC
#>  date     2022-02-27
#>  pandoc   1.19.2.4 @ /usr/bin/ (via rmarkdown)
#> 
#> ─ Packages ──────────────────────────────────────
#>  package      * version     date (UTC) lib source
#>  assertthat     0.2.1       2019-03-21 [1] CRAN (R 4.0.0)
#>  audubon        0.0.5       2022-02-06 [1] Github (paithiov909/audubon@431b00b)
#>  backports      1.4.1       2021-12-13 [1] CRAN (R 4.1.2)
#>  base64enc      0.1-3       2015-07-28 [1] CRAN (R 4.0.0)
#>  bit            4.0.4       2020-08-04 [1] CRAN (R 4.0.2)
#>  bit64          4.0.5       2020-08-30 [1] CRAN (R 4.0.2)
#>  broom        * 0.7.12      2022-01-28 [1] CRAN (R 4.1.2)
#>  cachem         1.0.6       2021-08-19 [1] CRAN (R 4.1.2)
#>  class          7.3-20      2022-01-13 [4] CRAN (R 4.1.2)
#>  cli            3.1.1       2022-01-20 [1] CRAN (R 4.1.2)
#>  codetools      0.2-18      2020-11-04 [4] CRAN (R 4.0.3)
#>  colorspace     2.0-2       2021-06-24 [1] CRAN (R 4.1.0)
#>  conflicted     1.1.0       2021-11-26 [1] CRAN (R 4.1.2)
#>  crayon         1.4.2       2021-10-29 [1] CRAN (R 4.1.1)
#>  curl           4.3.2       2021-06-23 [1] CRAN (R 4.1.0)
#>  data.table     1.14.2      2021-09-27 [1] CRAN (R 4.1.1)
#>  DBI            1.1.2       2021-12-20 [1] CRAN (R 4.1.2)
#>  dials        * 0.1.0       2022-01-31 [1] CRAN (R 4.1.2)
#>  DiceDesign     1.9         2021-02-13 [1] CRAN (R 4.1.2)
#>  digest         0.6.29      2021-12-01 [1] CRAN (R 4.1.2)
#>  distill        1.3         2021-10-13 [1] CRAN (R 4.1.2)
#>  doParallel     1.0.17      2022-02-07 [1] CRAN (R 4.1.2)
#>  downlit        0.4.0       2021-10-29 [1] CRAN (R 4.1.2)
#>  dplyr        * 1.0.7       2021-06-18 [1] CRAN (R 4.1.0)
#>  ellipsis       0.3.2       2021-04-29 [1] CRAN (R 4.0.5)
#>  embed          0.1.5       2021-11-24 [1] CRAN (R 4.1.2)
#>  evaluate       0.14        2019-05-28 [1] CRAN (R 4.0.0)
#>  fansi          1.0.2       2022-01-14 [1] CRAN (R 4.1.2)
#>  farver         2.1.0       2021-02-28 [1] CRAN (R 4.0.4)
#>  fastmap        1.1.0       2021-01-25 [1] CRAN (R 4.0.3)
#>  float          0.2-6       2021-09-20 [1] CRAN (R 4.1.2)
#>  foreach        1.5.2       2022-02-02 [1] CRAN (R 4.1.2)
#>  furrr          0.2.3       2021-06-25 [1] CRAN (R 4.1.2)
#>  future         1.23.0      2021-10-31 [1] CRAN (R 4.1.2)
#>  future.apply   1.8.1       2021-08-10 [1] CRAN (R 4.1.2)
#>  generics       0.1.2       2022-01-31 [1] CRAN (R 4.1.2)
#>  ggplot2      * 3.3.5       2021-06-25 [1] CRAN (R 4.1.0)
#>  globals        0.14.0      2020-11-22 [1] CRAN (R 4.1.2)
#>  glue           1.6.1       2022-01-22 [1] CRAN (R 4.1.2)
#>  gower          1.0.0       2022-02-03 [1] CRAN (R 4.1.2)
#>  GPfit          1.0-8       2019-02-08 [1] CRAN (R 4.1.2)
#>  gtable         0.3.0       2019-03-25 [1] CRAN (R 4.0.0)
#>  hardhat        0.2.0       2022-01-24 [1] CRAN (R 4.1.2)
#>  highr          0.9         2021-04-16 [1] CRAN (R 4.0.5)
#>  hms            1.1.1       2021-09-26 [1] CRAN (R 4.1.1)
#>  htmltools      0.5.2       2021-08-25 [1] CRAN (R 4.1.1)
#>  httpgd         1.3.0       2022-02-02 [1] CRAN (R 4.1.2)
#>  infer        * 1.0.0       2021-08-13 [1] CRAN (R 4.1.2)
#>  ipred          0.9-12      2021-09-15 [1] CRAN (R 4.1.2)
#>  irlba          2.3.5       2021-12-06 [1] CRAN (R 4.1.2)
#>  iterators      1.0.14      2022-02-05 [1] CRAN (R 4.1.2)
#>  jsonlite       1.7.3       2022-01-17 [1] CRAN (R 4.1.2)
#>  keras          2.7.0       2021-11-09 [1] CRAN (R 4.1.2)
#>  knitr          1.37        2021-12-16 [1] CRAN (R 4.1.2)
#>  labeling       0.4.2       2020-10-20 [1] CRAN (R 4.0.3)
#>  later          1.3.0       2021-08-18 [1] CRAN (R 4.1.2)
#>  lattice        0.20-45     2021-09-22 [1] CRAN (R 4.1.1)
#>  lava           1.6.10      2021-09-02 [1] CRAN (R 4.1.2)
#>  ldccr          0.0.6.900   2022-02-06 [1] Github (paithiov909/ldccr@b23ef2f)
#>  lgr            0.4.3       2021-09-16 [1] CRAN (R 4.1.2)
#>  lhs            1.1.3       2021-09-08 [1] CRAN (R 4.1.2)
#>  lifecycle      1.0.1       2021-09-24 [1] CRAN (R 4.1.1)
#>  listenv        0.8.0       2019-12-05 [1] CRAN (R 4.1.2)
#>  lubridate      1.8.0       2021-10-07 [1] CRAN (R 4.1.1)
#>  magrittr       2.0.2       2022-01-26 [1] CRAN (R 4.1.2)
#>  MASS           7.3-55      2022-01-13 [1] CRAN (R 4.1.2)
#>  Matrix         1.4-0       2021-12-08 [1] CRAN (R 4.1.2)
#>  memoise        2.0.1       2021-11-26 [1] CRAN (R 4.1.2)
#>  mlapi          0.1.0       2017-12-17 [1] CRAN (R 4.1.2)
#>  modeldata    * 0.1.1       2021-07-14 [1] CRAN (R 4.1.2)
#>  munsell        0.5.0       2018-06-12 [1] CRAN (R 4.0.0)
#>  nnet           7.3-17      2022-01-13 [4] CRAN (R 4.1.2)
#>  parallelly     1.30.0      2021-12-17 [1] CRAN (R 4.1.2)
#>  parsnip      * 0.1.7       2021-07-21 [1] CRAN (R 4.1.2)
#>  pillar         1.7.0       2022-02-01 [1] CRAN (R 4.1.2)
#>  pkgconfig      2.0.3       2019-09-22 [1] CRAN (R 4.0.0)
#>  plyr           1.8.6       2020-03-03 [1] CRAN (R 4.1.2)
#>  png            0.1-7       2013-12-03 [1] CRAN (R 4.1.2)
#>  pROC           1.18.0      2021-09-03 [1] CRAN (R 4.1.2)
#>  prodlim        2019.11.13  2019-11-17 [1] CRAN (R 4.1.2)
#>  purrr        * 0.3.4       2020-04-17 [1] CRAN (R 4.0.0)
#>  R.cache        0.15.0      2021-04-30 [1] CRAN (R 4.1.2)
#>  R.methodsS3    1.8.1       2020-08-26 [1] CRAN (R 4.1.2)
#>  R.oo           1.24.0      2020-08-26 [1] CRAN (R 4.1.2)
#>  R.utils        2.11.0      2021-09-26 [1] CRAN (R 4.1.2)
#>  R6             2.5.1       2021-08-19 [1] CRAN (R 4.1.1)
#>  Rcpp           1.0.8       2022-01-13 [1] CRAN (R 4.1.2)
#>  RcppMeCab      0.0.1.3.900 2022-02-06 [1] Github (paithiov909/RcppMeCab@154e75f)
#>  RcppParallel   5.1.5       2022-01-05 [1] CRAN (R 4.1.2)
#>  readr          2.1.1       2021-11-30 [1] CRAN (R 4.1.2)
#>  recipes      * 0.1.17      2021-09-27 [1] CRAN (R 4.1.2)
#>  reticulate     1.24        2022-01-26 [1] CRAN (R 4.1.2)
#>  RhpcBLASctl    0.21-247.1  2021-11-05 [1] CRAN (R 4.1.2)
#>  rlang        * 1.0.1       2022-02-03 [1] CRAN (R 4.1.2)
#>  rmarkdown      2.11        2021-09-14 [1] CRAN (R 4.1.1)
#>  rpart          4.1.16      2022-01-24 [4] CRAN (R 4.1.2)
#>  rsample      * 0.1.1       2021-11-08 [1] CRAN (R 4.1.2)
#>  rsparse        0.5.0       2021-11-30 [1] CRAN (R 4.1.2)
#>  rstudioapi     0.13        2020-11-12 [1] CRAN (R 4.0.3)
#>  scales       * 1.1.1       2020-05-11 [1] CRAN (R 4.0.0)
#>  sessioninfo    1.2.2       2021-12-06 [3] CRAN (R 4.1.2)
#>  stringi        1.7.6       2021-11-29 [1] CRAN (R 4.1.2)
#>  stringr        1.4.0       2019-02-10 [1] CRAN (R 4.0.0)
#>  styler         1.6.2       2021-09-23 [1] CRAN (R 4.1.2)
#>  survival       3.2-13      2021-08-24 [4] CRAN (R 4.1.1)
#>  systemfonts    1.0.3       2021-10-13 [1] CRAN (R 4.1.2)
#>  tensorflow     2.7.0       2021-11-09 [1] CRAN (R 4.1.2)
#>  text2vec     * 0.6         2020-02-18 [1] CRAN (R 4.1.2)
#>  textrecipes  * 0.4.1       2021-07-11 [1] CRAN (R 4.1.2)
#>  tfruns         1.5.0       2021-02-26 [1] CRAN (R 4.1.2)
#>  tibble       * 3.1.6       2021-11-07 [1] CRAN (R 4.1.2)
#>  tidymodels   * 0.1.4       2021-10-01 [1] CRAN (R 4.1.2)
#>  tidyr        * 1.2.0       2022-02-01 [1] CRAN (R 4.1.2)
#>  tidyselect     1.1.1       2021-04-30 [1] CRAN (R 4.0.5)
#>  timeDate       3043.102    2018-02-21 [1] CRAN (R 4.1.2)
#>  tune         * 0.1.6       2021-07-21 [1] CRAN (R 4.1.2)
#>  tzdb           0.2.0       2021-10-27 [1] CRAN (R 4.1.1)
#>  utf8           1.2.2       2021-07-24 [1] CRAN (R 4.1.0)
#>  uwot           0.1.11      2021-12-02 [1] CRAN (R 4.1.2)
#>  V8             4.0.0       2021-12-23 [1] CRAN (R 4.1.2)
#>  vctrs        * 0.3.8       2021-04-29 [1] CRAN (R 4.0.5)
#>  vroom          1.5.7       2021-11-30 [1] CRAN (R 4.1.2)
#>  whisker        0.4         2019-08-28 [1] CRAN (R 4.1.2)
#>  withr          2.4.3       2021-11-30 [1] CRAN (R 4.1.2)
#>  workflows    * 0.2.4       2021-10-12 [1] CRAN (R 4.1.2)
#>  workflowsets * 0.1.0       2021-07-22 [1] CRAN (R 4.1.2)
#>  xfun           0.29        2021-12-14 [1] CRAN (R 4.1.2)
#>  xgboost      * 1.5.2.1     2022-02-21 [1] CRAN (R 4.1.2)
#>  yaml           2.2.2       2022-01-25 [1] CRAN (R 4.1.2)
#>  yardstick    * 0.0.9       2021-11-22 [1] CRAN (R 4.1.2)
#>  zeallot        0.1.0       2018-01-28 [1] CRAN (R 4.1.2)
#> 
#>  [1] /content/workspace/renv/library/R-4.1/x86_64-pc-linux-gnu
#>  [2] /usr/local/lib/R/site-library
#>  [3] /usr/lib/R/site-library
#>  [4] /usr/lib/R/library
#> 
#> ─────────────────────────────────────────────────
```
