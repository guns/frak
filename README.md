# frak

frak transforms collections of strings into regular expressions for
matching those strings. The primary goal of this library is to
generate regular expressions from a known set of inputs which avoid
backtracking as much as possible.

## "Installation"

Add frak as a dependency to your `project.clj` file.

```clojure
[frak "0.1.2"]
```

## Usage

```clojure
user> (require 'frak)
nil
user> (frak/pattern ["foo" "bar" "baz" "quux"])
#"(?:ba[rz]|foo|quux)"
user> (frak/pattern ["Clojure" "Clojars" "ClojureScript"])
#"Cloj(?:ure(?:Script)?|ars)"
```

## How?

A frak pattern is constructed from a trie of characters. As characters
are added to it, meta data is stored in it's branches containing
information such as which branches are terminal and a record of
characters which have "visited" the branch.

During the rendering process frak will prefer branch characters that
have "visited" the most. In the example above, you will notice the
`ba[rz]` branch takes precedence over `foo` even though `"foo"` was
the first to enter the trie. This is because the character `\b` has
frequented the branch more than `\f` and `\q`. The example below
illustrates this behavior on the second character of each input.

```clojure
user> (frak/pattern ["bit" "bat" "ban" "bot" "bar" "box"])
#"b(?:a[tnr]|o[tx]|it)"
```

## Why?

[Here's](https://github.com/guns/vim-clojure-static/blob/249328ee659190babe2b14cd119f972b21b80538/syntax/clojure.vim#L91-L92)
why. Also because.

## And now for something completely different

Let's build a regular expression for matching any word in
`/usr/share/dict/words`.

```clojure
user> (require '[clojure.java.io :as io])
nil
user> (def words
           (-> (io/file "/usr/share/dict/words")
               io/reader
               line-seq))
#'user/words
user> (def word-re (frak/pattern words))
#'user/word-re
user> (every? #(re-matches word-re %) words)
true
```

The last two operations will take a moment since there are about
235,886 words to consider.

You can view the full expression
[here](https://gist.github.com/noprompt/6106573/raw/fcb683834bb2e171618ca91bf0b234014b5b957d/word-re.clj)
(it's approximately `1.5M`!).

## Benchmarks

```clojure
(use 'criterium.core)

(def words
  (-> (io/file "/usr/share/dict/words")
      io/reader
      line-seq))

(defn naive-pattern
  "Create a naive regular expression pattern for matching every string
   in strs."
  [strs]
  (->> strs
       (clojure.string/join "|")
       (format "(?:%s)")
       re-pattern))

;; Shuffle 10000 words and build a naive and frak pattern from them.
(def ws (shuffle (take 10000 words)))
(def n-pat (naive-pattern ws))
(def f-pat (frak/pattern ws))

;; Verify the naive pattern matches everything it was constructed from.
(every? #(re-matches n-pat %) ws)
;; => true

;; Shuffle the words again since the naive pattern is built in the
;; same order as it's inputs.
(def ws' (shuffle ws))

;;;; Benchmarks

;; Naive pattern

(bench (doseq [w ws'] (re-matches n-pat w)))
;;             Execution time mean : 1.499489 sec
;;    Execution time std-deviation : 181.365166 ms
;;   Execution time lower quantile : 1.337817 sec ( 2.5%)
;;   Execution time upper quantile : 1.828733 sec (97.5%)

;; frak pattern

(bench (doseq [w ws'] (re-matches f-pat w)))
;;             Execution time mean : 155.515855 ms
;;    Execution time std-deviation : 5.663346 ms
;;   Execution time lower quantile : 148.168855 ms ( 2.5%)
;;   Execution time upper quantile : 164.164294 ms (97.5%)
```
