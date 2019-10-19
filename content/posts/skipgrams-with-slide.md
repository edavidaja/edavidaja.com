---
title: "skipgrams with {slide}"
date: 2019-10-18T15:51:58-04:00
draft: false
---

In addition to the the essential [Tidy Text Mining With
R](https://www.tidytextmining.com/), [Julia Silge’s personal
blog](https://juliasilge.com/blog/) is a frequent reference for me when
doing tidy text analysis. Yesterday, I was working on detecting unusual
bigrams in a corpus of restaurant reviews, and remembered that Julia had
written a `slide_windows()` function to simplify preparing skipgrams for
`widyr::pairwise_pmi()`.

Here’s the code original code to compute skip grams:

```r
library(tidytext)
library(dplyr)
library(purrr)
library(slide)

slide_windows <- function(tbl, doc_var, window_size) {
  # each word gets a skipgram (window_size words) starting on the first
  # e.g. skipgram 1 starts on word 1, skipgram 2 starts on word 2
  
  each_total <- tbl %>% 
    group_by(!!doc_var) %>% 
    mutate(doc_total = n(),
            each_total = pmin(doc_total, window_size, na.rm = TRUE)) %>%
    pull(each_total)
  
  rle_each <- rle(each_total)
  counts <- rle_each[["lengths"]]
  counts[rle_each$values != window_size] <- 1
  
  # each word get a skipgram window, starting on the first
  # account for documents shorter than window
  id_counts <- rep(rle_each$values, counts)
  window_id <- rep(seq_along(id_counts), id_counts)
  
  # within each skipgram, there are window_size many offsets
  indexer <- (seq_along(rle_each[["values"]]) - 1) %>%
    map2(rle_each[["values"]] - 1,
          ~ seq.int(.x, .x + .y)) %>% 
    map2(counts, ~ rep(.x, .y)) %>%
    flatten_int() +
    window_id
  
  tbl[indexer, ] %>%
    bind_cols(data_frame(window_id)) %>%
    group_by(window_id) %>%
    filter(n_distinct(!!doc_var) == 1) %>%
    ungroup
}
```


Here’s what it returns on a trivial data set:

```r
test <- tibble(
  speaker = c("banner", "strange"),
  text = c("thanos is coming", "who is thanos")
  )

test %>% 
  unnest_tokens(word, text) %>% 
  slide_windows(quo(speaker), 2)

## # A tibble: 8 x 3
##   speaker word   window_id
##   <chr>   <chr>      <int>
## 1 banner  thanos         1
## 2 banner  is             1
## 3 banner  is             2
## 4 banner  coming         2
## 5 strange who            4
## 6 strange is             4
## 7 strange is             5
## 8 strange thanos         5
```

The function name gave me an idea: I could use Davis Vaughan’s
[`{slide}`](https://davisvaughan.github.io/slide/index.html) as an
another way to compute the necessary sliding windows. Slide is
especially exciting if you’ve ever had to do rolling computations with
business calendars, but it is, as advertised, quite general-purpose.

Here’s the function with `slide()`:

```r
new_slide_windows <- function(tbl, doc_var, window_size) {
  
  window_size <- window_size - 1

  grams <- slide(tbl, ~.x, .after = window_size, .step = 1, .complete = TRUE)
  
  # because .complete returns NULL if a group is not complete
  # and I am matching the numbering scheme of the original function
  safe_mutate <- safely(mutate)
  
  out <- map2(grams, 1:length(grams), ~safe_mutate(.x, window_id = .y))
  
  out %>%
    transpose() %>% 
    pluck("result") %>% 
    compact() %>%
    # this discards any data frames with varying speakers
    discard(~length(unique(.x[[doc_var]])) > 1) %>% 
    bind_rows()
}
```

And it returns the same results:

```r
test %>% 
  unnest_tokens(word, text) %>% 
  new_slide_windows("speaker", 2)

## # A tibble: 8 x 3
##   speaker word   window_id
##   <chr>   <chr>      <int>
## 1 banner  thanos         1
## 2 banner  is             1
## 3 banner  is             2
## 4 banner  coming         2
## 5 strange who            4
## 6 strange is             4
## 7 strange is             5
## 8 strange thanos         5
```

Rearranged slightly:

```r
new_slide_windows <- function(tbl, doc_var, window_size) {
  
  window_size <- window_size - 1
  
  grams <- slide(tbl, ~.x, .after = window_size, .step = 1, .complete = TRUE) %>%
    compact()  %>% 
    discard(~length(unique(.x[[doc_var]])) > 1) 
  
  out <- map2(grams, 1:length(grams), ~mutate(.x, window_id = .y))
  
  out %>%
    bind_rows()
}
```

{tidytext} has gained a skip-gram tokenizer since the initial blog post
was published, so none of this is strictly necessary, but this blog
wasn’t going to start itself!
