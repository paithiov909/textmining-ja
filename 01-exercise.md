---
title: "NLP100ノック：準備運動・UNIXコマンド・正規表現"
author: "paithiov909"
date: "2022-03-03"
output: html_document
---

# NLP100ノック：準備運動・UNIXコマンド・正規表現



## 準備運動

コーディングの方針として、文字はなるべくリストのまま持っておいて最後に`unlist`する感じにしています。

よく知られているように、Rにおける文字列は、実体としては、文字列ベクトル（character vector）です。つまり、たとえば`str <- "aaa"`などとしても、このstrは長さが1のベクトルになります。
このため、R言語を触っている人たちのあいだでは、文字列ベクトルの一要素としてのいわゆる「文字列」のことは、string scalarとか単にstringなどと呼ばれます（たぶん）。

こうした事情から、R言語レベルで「文字列（string scalar）」の文字ごとに処理を繰り返そうとすると混乱をまねきやすいため、「文字」に対して処理をまわす場合はリスト（list of strings）持ちにしたほうがわかりやすいです（個人の感想）。

また、基本的にbaseの文字列処理よりもstringrを優先して使うようにしています（R>=4.1.0が必要だが、baseの文字列処理をstringiをラップした同名の関数でマスクできる[stringx](https://stringx.gagolewski.com/)というパッケージがあって、これを使うとタイプする文字数やヘルプを確認する頻度を減らせる）。


```r
## ncharとpaste0だけマスクしておく
nchar <- stringr::str_length
paste0 <- stringr::str_c
```

### 00. 文字列の逆順


```r
stringr::str_split("stressed", pattern = "") %>%
  purrr::map(~ rev(.)) %>%
  unlist() %>%
  paste0(collapse = "")
#> [1] "desserts"
```

### 01. 「パタトクカシーー」


```r
stringr::str_split("パタトクカシーー", pattern = "") %>%
  purrr::map(~ purrr::pluck(.[c(TRUE, FALSE)])) %>%
  unlist() %>%
  paste0(collapse = "")
#> [1] "パトカー"
```

### 02. 「パトカー」＋「タクシー」＝「パタトクカシーー」


```r
list("パトカー", "タクシー") %>%
  purrr::map(~ stringr::str_split(., pattern = "")) %>%
  purrr::flatten() %>%
  purrr::pmap(~ paste0(.x, .y, collapse = "")) %>%
  unlist() %>%
  paste0(collapse = "")
#> [1] "パタトクカシーー"
```

### 03. 円周率


```r
stringr::str_split("Now I need a drink, alcoholic of course, after the heavy lectures involving quantum mechanics.", pattern = " ") %>%
  purrr::flatten() %>%
  purrr::map(~ stringr::str_count(., pattern = "[:alpha:]")) %>%
  unlist()
#>  [1] 3 1 4 1 5 9 2 6 5 3 5 8 9 7 9
```

### 04. 元素記号


```r
stringr::str_split("Hi He Lied Because Boron Could Not Oxidize Fluorine. New Nations Might Also Sign Peace Security Clause. Arthur King Can.", pattern = " ") %>%
  purrr::flatten() %>%
  purrr::imap(~
  dplyr::if_else(
    .y %in% c(1, 5, 6, 7, 8, 9, 15, 16, 19),
    stringr::str_sub(.x, 1, 1),
    stringr::str_sub(.x, 1, 2)
  )) %>%
  purrr::imap(function(x, i) {
    names(x) <- i
    return(x)
  }) %>%
  unlist()
#>    1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17 
#>  "H" "He" "Li" "Be"  "B"  "C"  "N"  "O"  "F" "Ne" "Na" "Mi" "Al" "Si"  "P"  "S" "Cl" 
#>   18   19   20 
#> "Ar"  "K" "Ca"
```

### 05. n-gram


```r
ngram <- function(x, n = 2, sep = " ") {
  stopifnot(is.character(x))
  ## 先例がみんな`embed`を使っているが、ここでは使わない

  tokens <- unlist(stringr::str_split(x, pattern = sep))
  len <- length(tokens)

  if (len < n) {
    res <- character(0)
  } else {
    res <- sapply(1:max(1, len - n + 1), function(i) {
      paste0(tokens[i:min(len, i + n - 1)], collapse = " ")
    })
  }

  return(res)
}
ngram("I am an NLPer")
#> [1] "I am"     "am an"    "an NLPer"
```

### 06. 集合

回答略

### 07. テンプレートによる文生成

回答略

### 08. 暗号文


```r
cipher <- function(str) {
  f <- purrr::as_mapper(~ 219 - .)
  v <- stringr::str_split(str, pattern = "", simplify = TRUE)
  res <- sapply(v[1, ], function(char) {
    dplyr::if_else(
      stringr::str_detect(char, "[:lower:]"),
      char %>%
        charToRaw() %>%
        as.integer() %>%
        f() %>%
        as.raw() %>%
        rawToChar(),
      char
    )
  })
  return(paste0(res, collapse = ""))
}
cipher("I couldn't believe that I could actually understand what I was reading : the phenomenal power of the human mind.")
#> [1] "I xlfowm'g yvorvev gszg I xlfow zxgfzoob fmwvihgzmw dszg I dzh ivzwrmt : gsv ksvmlnvmzo kldvi lu gsv sfnzm nrmw."
```

### 09. Typoglycemia


```r
typoglycemia <- function(str) {
  f <- function(char) {
    subset <- stringr::str_sub(char, 2, nchar(char) - 1) %>%
      stringr::str_split(pattern = "") %>%
      purrr::flatten() %>%
      sample()
    res <- paste0(
      c(
        stringr::str_sub(char, 1, 1),
        subset,
        stringr::str_sub(char, nchar(char), nchar(char))
      ),
      collapse = ""
    )
    return(res)
  }
  res <- stringr::str_split(str, pattern = " ") %>%
    purrr::flatten() %>%
    purrr::map(~
    dplyr::if_else(
      nchar(stringr::str_subset(., "[:alpha:]|:")) <= 4,
      .,
      f(.)
    ))
  return(paste0(res, collapse = " "))
}
typoglycemia("I couldn't believe that I could actually understand what I was reading : the phenomenal power of the human mind.")
#> [1] "I clduon't beievle that I cloud autalcly udsenrtand what I was rdianeg : the pnhmeeoanl peowr of the hmaun mndi."
```

## UNIXコマンド

確認はやりません。~~だってWindowsだもん~~

### 10~15

素のテキストとして読んでもしょうがないので、以下のようなこと雰囲気でやります。

- 10. 行数のカウント
- 11. タブをスペースに置換
- 14. 先頭からN行を出力
- 15. 末尾のN行を出力

以下の２つはやりませんが、たぶん`fread(temp, select = c(1, 2))`みたいな感じで取れます。

- 12. 1列目をcol1.txtに，2列目をcol2.txtに保存
- 13. col1.txtとcol2.txtをマージ


```r
txt <-
  data.table::fread(
    "https://nlp100.github.io/data/popular-names.txt",
    sep = "\t",
    quote = "",
    header = FALSE,
    col.names = c("name", "sex", "num_of_people", "year"),
    colClasses = list("character" = 1, "character" = 2, "integer" = 3, "integer" = 4),
    data.table = FALSE
  )

nrow(txt)
#> [1] 2780
```


```r
head(txt, 3)
#>   name sex num_of_people year
#> 1 Mary   F          7065 1880
#> 2 Anna   F          2604 1880
#> 3 Emma   F          2003 1880
```


```r
tail(txt, 3)
#>       name sex num_of_people year
#> 2778 Lucas   M         12585 2018
#> 2779 Mason   M         12435 2018
#> 2780 Logan   M         12352 2018
```

### 16. ファイルをN分割する


```r
split(txt, sort(rank(row.names(txt)) %% 5)) %>%
  purrr::map(~ head(.)) %>%
  print()
#> $`0`
#>        name sex num_of_people year
#> 1      Mary   F          7065 1880
#> 2      Anna   F          2604 1880
#> 3      Emma   F          2003 1880
#> 4 Elizabeth   F          1939 1880
#> 5    Minnie   F          1746 1880
#> 6  Margaret   F          1578 1880
#> 
#> $`1`
#>       name sex num_of_people year
#> 557 Joseph   M          3844 1907
#> 558  Frank   M          2943 1907
#> 559 Edward   M          2576 1907
#> 560  Henry   M          2203 1907
#> 561   Mary   F         18665 1908
#> 562  Helen   F          8439 1908
#> 
#> $`2`
#>         name sex num_of_people year
#> 1113    John   M         47499 1935
#> 1114 William   M         40198 1935
#> 1115 Richard   M         33945 1935
#> 1116 Charles   M         29983 1935
#> 1117  Donald   M         29661 1935
#> 1118  George   M         18559 1935
#> 
#> $`3`
#>         name sex num_of_people year
#> 1669  Sandra   F         21619 1963
#> 1670 Cynthia   F         21593 1963
#> 1671 Michael   M         83782 1963
#> 1672    John   M         78625 1963
#> 1673   David   M         78467 1963
#> 1674   James   M         71322 1963
#> 
#> $`4`
#>           name sex num_of_people year
#> 2225  Samantha   F         25645 1991
#> 2226     Sarah   F         25225 1991
#> 2227 Stephanie   F         22774 1991
#> 2228  Jennifer   F         20673 1991
#> 2229 Elizabeth   F         20392 1991
#> 2230     Emily   F         20308 1991
```

### 17. １列目の文字列の異なり

省略

### 18. 各行を3コラム目の数値の降順にソート


```r
txt %>%
  dplyr::arrange(desc(num_of_people)) %>%
  head()
#>      name sex num_of_people year
#> 1   Linda   F         99689 1947
#> 2   Linda   F         96211 1948
#> 3   James   M         94757 1947
#> 4 Michael   M         92704 1957
#> 5  Robert   M         91640 1947
#> 6   Linda   F         91016 1949
```

### 19. 各行の1コラム目の文字列の出現頻度を求め，出現頻度の高い順に並べる


```r
purrr::map_dfr(txt$name, function(name) {
  stringr::str_split(name, pattern = "", simplify = TRUE) %>%
    t() %>%
    as.data.frame(stringsAsFactors = FALSE)
}) %>%
  dplyr::rename(string = V1) %>%
  dplyr::group_by(string) %>%
  dplyr::count(string, sort = TRUE) %>%
  dplyr::ungroup() %>%
  head()
#> # A tibble: 6 x 2
#>   string     n
#>   <chr>  <int>
#> 1 a       2194
#> 2 e       1554
#> 3 r       1270
#> 4 i       1183
#> 5 h       1018
#> 6 l        943
```

## 正規表現

自然言語処理とはいったい

### 20. JSONデータの読み込み


```r
jsonfile <- (function() {
  ## ファイルコネクションの閉じ忘れ防止に「閉包」に閉じ込めている
  ## ただし、実は.gz, .bz2, .xz, .zipファイルはリモートのファイルであってもread_linesで直接読むことが可能なので、
  ## 本当はこんなことする必要はない
  temp <- tempfile(fileext = ".gz")
  download.file("https://nlp100.github.io/data/jawiki-country.json.gz", temp, quiet = TRUE)
  con <- gzfile(description = temp, open = "rb")
  on.exit(close(con))
  readr::read_lines(con, locale = readr::locale(encoding = "UTF-8")) %>%
    purrr::map_dfr(~
    jsonlite::fromJSON(.))
})()

jsonfile %>%
  dplyr::filter(title == "イギリス") %>%
  dplyr::pull(text) %>%
  dplyr::glimpse() # 長いので
#>  chr "{{redirect|UK}}\n{{redirect|英国|春秋時代の諸侯国|英 (春秋)}}\n{{Otheruses|ヨーロッパの国|長崎県・熊本県の郷土"| __truncated__
```

### 21. カテゴリ名を含む行を抽出


```r
lines <- jsonfile %>%
  dplyr::filter(title == "イギリス") %>%
  dplyr::pull(text) %>%
  readr::read_lines() %>%
  stringr::str_subset(stringr::fixed("[[Category:"))
head(lines)
#> [1] "[[Category:イギリス|*]]"         "[[Category:イギリス連邦加盟国]]"
#> [3] "[[Category:英連邦王国|*]]"       "[[Category:G8加盟国]]"          
#> [5] "[[Category:欧州連合加盟国|元]]"  "[[Category:海洋国家]]"
```

以下、回答略

## セッション情報


```r
sessioninfo::session_info()
#> - Session info -----------------------------------------------------------------------
#>  setting  value
#>  version  R version 4.1.2 (2021-11-01)
#>  os       Windows 10 x64 (build 19043)
#>  system   x86_64, mingw32
#>  ui       RStudio
#>  language (EN)
#>  collate  Japanese_Japan.932
#>  ctype    Japanese_Japan.932
#>  tz       Asia/Tokyo
#>  date     2022-03-03
#>  rstudio  2022.02.0+443 Prairie Trillium (desktop)
#>  pandoc   2.17.1.1 @ C:/Program Files/RStudio/bin/quarto/bin/ (via rmarkdown)
#> 
#> - Packages ---------------------------------------------------------------------------
#>  ! package             * version     date (UTC) lib source
#>    abind                 1.4-5       2016-07-21 [1] CRAN (R 4.1.1)
#>    askpass               1.1         2019-01-13 [1] CRAN (R 4.1.2)
#>    assertthat            0.2.1       2019-03-21 [1] CRAN (R 4.1.2)
#>    audubon               0.1.1       2022-02-14 [1] CRAN (R 4.1.2)
#>    backports             1.4.1       2021-12-13 [1] CRAN (R 4.1.2)
#>    bit                   4.0.4       2020-08-04 [1] CRAN (R 4.1.2)
#>    bit64                 4.0.5       2020-08-30 [1] CRAN (R 4.1.2)
#>    broom                 0.7.12      2022-01-28 [1] CRAN (R 4.1.2)
#>    cachem                1.0.6       2021-08-19 [1] CRAN (R 4.1.2)
#>    car                   3.0-12      2021-11-06 [1] CRAN (R 4.1.2)
#>    carData               3.0-5       2022-01-06 [1] CRAN (R 4.1.2)
#>    class                 7.3-19      2021-05-03 [2] CRAN (R 4.1.2)
#>    cli                   3.2.0       2022-02-14 [1] CRAN (R 4.1.2)
#>    codetools             0.2-18      2020-11-04 [2] CRAN (R 4.1.2)
#>    colorspace            2.0-3       2022-02-21 [1] CRAN (R 4.1.2)
#>    crayon                1.5.0       2022-02-14 [1] CRAN (R 4.1.2)
#>    curl                  4.3.2       2021-06-23 [1] CRAN (R 4.1.2)
#>    data.table            1.14.2      2021-09-27 [1] CRAN (R 4.1.2)
#>    DBI                   1.1.2       2021-12-20 [1] CRAN (R 4.1.2)
#>    dials                 0.1.0       2022-01-31 [1] CRAN (R 4.1.2)
#>    DiceDesign            1.9         2021-02-13 [1] CRAN (R 4.1.2)
#>    digest                0.6.29      2021-12-01 [1] CRAN (R 4.1.2)
#>    doParallel            1.0.17      2022-02-07 [1] CRAN (R 4.1.2)
#>    dplyr                 1.0.8       2022-02-08 [1] CRAN (R 4.1.2)
#>    ellipsis              0.3.2       2021-04-29 [1] CRAN (R 4.1.2)
#>    evaluate              0.15        2022-02-18 [1] CRAN (R 4.1.2)
#>    fansi                 1.0.2       2022-01-14 [1] CRAN (R 4.1.2)
#>    fastmap               1.1.0       2021-01-25 [1] CRAN (R 4.1.2)
#>    fastmatch             1.1-3       2021-07-23 [1] CRAN (R 4.1.1)
#>    float                 0.2-6       2021-09-20 [1] CRAN (R 4.1.1)
#>    foreach               1.5.2       2022-02-02 [1] CRAN (R 4.1.2)
#>    furrr                 0.2.3       2021-06-25 [1] CRAN (R 4.1.2)
#>    future                1.24.0      2022-02-19 [1] CRAN (R 4.1.2)
#>    future.apply          1.8.1       2021-08-10 [1] CRAN (R 4.1.2)
#>    generics              0.1.2       2022-01-31 [1] CRAN (R 4.1.2)
#>    ggdendro              0.1.23      2022-02-16 [1] CRAN (R 4.1.2)
#>    ggplot2               3.3.5       2021-06-25 [1] CRAN (R 4.1.2)
#>    ggpubr                0.4.0       2020-06-27 [1] CRAN (R 4.1.2)
#>    ggrepel               0.9.1       2021-01-15 [1] CRAN (R 4.1.2)
#>    ggsignif              0.6.3       2021-09-09 [1] CRAN (R 4.1.2)
#>    glmnet                4.1-3       2021-11-02 [1] CRAN (R 4.1.2)
#>    globals               0.14.0      2020-11-22 [1] CRAN (R 4.1.1)
#>    glue                  1.6.2       2022-02-24 [1] CRAN (R 4.1.2)
#>    gower                 1.0.0       2022-02-03 [1] CRAN (R 4.1.2)
#>    GPfit                 1.0-8       2019-02-08 [1] CRAN (R 4.1.2)
#>    gridExtra             2.3         2017-09-09 [1] CRAN (R 4.1.2)
#>    gtable                0.3.0       2019-03-25 [1] CRAN (R 4.1.2)
#>    hardhat               0.2.0       2022-01-24 [1] CRAN (R 4.1.2)
#>    hms                   1.1.1       2021-09-26 [1] CRAN (R 4.1.2)
#>    htmltools             0.5.2       2021-08-25 [1] CRAN (R 4.1.2)
#>    httr                  1.4.2       2020-07-20 [1] CRAN (R 4.1.2)
#>    infer                 1.0.0       2021-08-13 [1] CRAN (R 4.1.2)
#>    ipred                 0.9-12      2021-09-15 [1] CRAN (R 4.1.2)
#>    iterators             1.0.14      2022-02-05 [1] CRAN (R 4.1.2)
#>    jsonlite              1.8.0       2022-02-22 [1] CRAN (R 4.1.2)
#>    knitr                 1.37        2021-12-16 [1] CRAN (R 4.1.2)
#>    lattice               0.20-45     2021-09-22 [2] CRAN (R 4.1.2)
#>    lava                  1.6.10      2021-09-02 [1] CRAN (R 4.1.2)
#>    LDAvis                0.3.2       2015-10-24 [1] CRAN (R 4.1.2)
#>    ldccr                 0.0.6.900   2022-02-05 [1] Github (paithiov909/ldccr@b23ef2f)
#>    lgr                   0.4.3       2021-09-16 [1] CRAN (R 4.1.2)
#>    lhs                   1.1.4       2022-02-20 [1] CRAN (R 4.1.2)
#>    LiblineaR             2.10-12     2021-03-02 [1] CRAN (R 4.1.2)
#>    lifecycle             1.0.1       2021-09-24 [1] CRAN (R 4.1.2)
#>    listenv               0.8.0       2019-12-05 [1] CRAN (R 4.1.2)
#>    lubridate             1.8.0       2021-10-07 [1] CRAN (R 4.1.2)
#>    magrittr            * 2.0.2       2022-01-26 [1] CRAN (R 4.1.2)
#>    MASS                  7.3-54      2021-05-03 [2] CRAN (R 4.1.2)
#>    Matrix                1.3-4       2021-06-01 [2] CRAN (R 4.1.2)
#>    memoise               2.0.1       2021-11-26 [1] CRAN (R 4.1.2)
#>    mlapi                 0.1.0       2017-12-17 [1] CRAN (R 4.1.2)
#>    munsell               0.5.0       2018-06-12 [1] CRAN (R 4.1.2)
#>    nnet                  7.3-16      2021-05-03 [2] CRAN (R 4.1.2)
#>    nsyllable             1.0.1       2022-02-28 [1] CRAN (R 4.1.2)
#>    openssl               1.4.6       2021-12-19 [1] CRAN (R 4.1.2)
#>    parallelly            1.30.0      2021-12-17 [1] CRAN (R 4.1.2)
#>    parsnip               0.1.7       2021-07-21 [1] CRAN (R 4.1.2)
#>    pillar                1.7.0       2022-02-01 [1] CRAN (R 4.1.2)
#>    pkgconfig             2.0.3       2019-09-22 [1] CRAN (R 4.1.2)
#>    plyr                  1.8.6       2020-03-03 [1] CRAN (R 4.1.2)
#>    png                   0.1-7       2013-12-03 [1] CRAN (R 4.1.1)
#>    pROC                  1.18.0      2021-09-03 [1] CRAN (R 4.1.2)
#>    prodlim               2019.11.13  2019-11-17 [1] CRAN (R 4.1.2)
#>    proxyC                0.2.4       2021-12-10 [1] CRAN (R 4.1.2)
#>    purrr                 0.3.4       2020-04-17 [1] CRAN (R 4.1.2)
#>    quanteda              3.2.1       2022-03-01 [1] CRAN (R 4.1.2)
#>    quanteda.textmodels   0.9.4       2021-04-06 [1] CRAN (R 4.1.2)
#>    quanteda.textplots    0.94        2021-04-06 [1] CRAN (R 4.1.2)
#>    quanteda.textstats    0.95        2021-11-24 [1] CRAN (R 4.1.2)
#>    R.cache               0.15.0      2021-04-30 [1] CRAN (R 4.1.2)
#>    R.methodsS3           1.8.1       2020-08-26 [1] CRAN (R 4.1.1)
#>    R.oo                  1.24.0      2020-08-26 [1] CRAN (R 4.1.1)
#>    R.utils               2.11.0      2021-09-26 [1] CRAN (R 4.1.2)
#>    R6                    2.5.1       2021-08-19 [1] CRAN (R 4.1.2)
#>    Rcpp                  1.0.8       2022-01-13 [1] CRAN (R 4.1.2)
#>    RcppMeCab             0.0.1.3.900 2022-01-17 [1] local
#>  D RcppParallel          5.1.5       2022-01-05 [1] CRAN (R 4.1.2)
#>    RcppProgress          0.4.2       2020-02-06 [1] CRAN (R 4.1.2)
#>    readr                 2.1.2       2022-01-30 [1] CRAN (R 4.1.2)
#>    recipes               0.2.0       2022-02-18 [1] CRAN (R 4.1.2)
#>    rematch2              2.1.2       2020-05-01 [1] CRAN (R 4.1.2)
#>    reticulate            1.24        2022-01-26 [1] CRAN (R 4.1.2)
#>    RhpcBLASctl           0.21-247.1  2021-11-05 [1] CRAN (R 4.1.2)
#>    rlang                 1.0.1       2022-02-03 [1] CRAN (R 4.1.2)
#>    rmarkdown             2.11        2021-09-14 [1] CRAN (R 4.1.2)
#>    rpart                 4.1-15      2019-04-12 [2] CRAN (R 4.1.2)
#>    rsample               0.1.1       2021-11-08 [1] CRAN (R 4.1.2)
#>    rsparse               0.5.0       2021-11-30 [1] CRAN (R 4.1.2)
#>    RSpectra              0.16-0      2019-12-01 [1] CRAN (R 4.1.2)
#>    rstatix               0.7.0       2021-02-13 [1] CRAN (R 4.1.2)
#>    rstudioapi            0.13        2020-11-12 [1] CRAN (R 4.1.2)
#>    rtweet                0.7.0       2020-01-08 [1] CRAN (R 4.1.2)
#>    scales                1.1.1       2020-05-11 [1] CRAN (R 4.1.2)
#>    sessioninfo           1.2.2       2021-12-06 [1] CRAN (R 4.1.2)
#>    shape                 1.4.6       2021-05-19 [1] CRAN (R 4.1.1)
#>    SparseM               1.81        2021-02-18 [1] CRAN (R 4.1.1)
#>    stopwords             2.3         2021-10-28 [1] CRAN (R 4.1.2)
#>    stringi               1.7.6       2021-11-29 [1] CRAN (R 4.1.2)
#>    stringr               1.4.0       2019-02-10 [1] CRAN (R 4.1.2)
#>    styler                1.6.2       2021-09-23 [1] CRAN (R 4.1.2)
#>    survival              3.2-13      2021-08-24 [2] CRAN (R 4.1.2)
#>    text2vec              0.6         2020-02-18 [1] CRAN (R 4.1.2)
#>    textmineR             3.0.5       2021-06-28 [1] CRAN (R 4.1.2)
#>    textrecipes           0.4.1       2021-07-11 [1] CRAN (R 4.1.2)
#>    tibble                3.1.6       2021-11-07 [1] CRAN (R 4.1.2)
#>    tidymodels            0.1.4       2021-10-01 [1] CRAN (R 4.1.2)
#>    tidyr                 1.2.0       2022-02-01 [1] CRAN (R 4.1.2)
#>    tidyselect            1.1.2       2022-02-21 [1] CRAN (R 4.1.2)
#>    timeDate              3043.102    2018-02-21 [1] CRAN (R 4.1.1)
#>    tune                  0.1.6       2021-07-21 [1] CRAN (R 4.1.2)
#>    tzdb                  0.2.0       2021-10-27 [1] CRAN (R 4.1.2)
#>    umap                  0.2.7.0     2020-11-04 [1] CRAN (R 4.1.2)
#>    utf8                  1.2.2       2021-07-24 [1] CRAN (R 4.1.2)
#>    V8                    4.1.0       2022-02-06 [1] CRAN (R 4.1.2)
#>    vctrs                 0.3.8       2021-04-29 [1] CRAN (R 4.1.2)
#>    viridis               0.6.2       2021-10-13 [1] CRAN (R 4.1.2)
#>    viridisLite           0.4.0       2021-04-13 [1] CRAN (R 4.1.2)
#>    vroom                 1.5.7       2021-11-30 [1] CRAN (R 4.1.2)
#>    withr                 2.4.3       2021-11-30 [1] CRAN (R 4.1.2)
#>    workflows             0.2.4       2021-10-12 [1] CRAN (R 4.1.2)
#>    workflowsets          0.1.0       2021-07-22 [1] CRAN (R 4.1.2)
#>    xfun                  0.29        2021-12-14 [1] CRAN (R 4.1.2)
#>    yaml                  2.3.5       2022-02-21 [1] CRAN (R 4.1.2)
#>    yardstick             0.0.9       2021-11-22 [1] CRAN (R 4.1.2)
#> 
#>  [1] C:/R/win-library/4.1
#>  [2] C:/Program Files/R/R-4.1.2/library
#> 
#>  D -- DLL MD5 mismatch, broken installation.
#> 
#> --------------------------------------------------------------------------------------
```

