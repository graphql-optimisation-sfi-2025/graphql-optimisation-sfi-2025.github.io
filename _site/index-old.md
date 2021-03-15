<h2>
Full-text search <br />
<span class='fragment'> index </span> <br />
inside-out</h2>

<hr/>

#### Maciek Rząsa
[@mjrzasa](https://twitter.com/mjrzasa)

Rzeszów Ruby User Group, 2.04.2019

::::

<h2 style='margin-top: -300px'> Find a cat </h2>

::::

<section
  data-background-image="img/rabbit_poziomo.png"
  style='min-height=50% important!'
>

<!--div style='margin-left: -900px; margin-top: -300px;' -->
<div class='fragment image-overlay' style='margin-top: -300px;'>
<h2> Find a rabbit </h2>
</div>

</section>


::::
## Find a cat

```
  docs = [
    "A kitten is a juvenile cat.
    After being born, kittens are totally dependent"

    "A dog is an uneducated creature."

    "Vindication is a part of every business."

    "Kites are great toys for cats."
  ]
```

```
# bad ideas
docs.find { |d| d.include?("cat") }
docs.find { |d| d.match(/^cats?$/) }
SELECT * FROM documents WHERE description LIKE %cat%;
```
::::

```
class SearchIndex
  def index(id, text)
  end

  def search(text)
  end
end
```

:::

<!-- tokenize - add -->
```
class SearchIndex
  def index(id, text)
    documents[id] = content
    tokens = tokenize(text)

    tokens.each do |token|
      if index[token]
        index[token] << id
      else
        index[token] = [id]
      end
    end
  end

  private
  def tokenize(text)
    text.split
  end
end

```
:::
<!-- tokenize - search -->
```
class SearchIndex
  def add(id, text)
    documents[id] = content
    tokens = tokenize(text)

    tokens.each do |token|
      if index[token]
        index[token] << id
      else
        index[token] = [id]
      end
    end
  end

  def search(text)
    tokens = tokenize(text)
    ids = index.fetch_values(*tokens)

    documents.fetch_values(*ids)
  end
end
```

:::
<!-- normalize -->
```
class SearchIndex
  def add(id, text)
    documents[id] = content
    tokens = normalize(tokenize(text))

    #...
  end

  def search(text)
    tokens = normalize(tokenize(text))
    ids = index.fetch_values(*tokens)

    documents.fetch_values(*ids)
  end

  def normalize(tokens)
    tokens.map { |t| t.downcase }
  end
end
```

:::
<!-- analyze -->
```
  def add(id, text)
    documents[id] = content
    tokens = analyze(text)
    #...
  def search(text)
    tokens = analyze(text)
    #...

  def analyze(text)
    normalize(tokenize(text))
  end
```

:::

<!-- filter -->
```
  def analyze(text)
    filter(normalize(tokenize(text)))
  end

  def filter(tokens)
    tokens.reject{ |t| t.match(/^\W$/) }
  end
```

:::
## Analyze
```
  "A kitten is a juvenile cat.
  After being born, kittens are totally dependent"
  =>
  ["a", "kitten", "is", "a", "juvenile",
  "cat", "after", "being", "born",
  "kittens", "are", "totally", "dependent"]

  ["kitten", "juvenile", "cat", "after", "born",
  "kittens", "totally", "dependent"]
```

::::


<section
  data-background-image="img/kitten.jpg"
  style='min-height=100% important!'
>

<div class='image-overlay' style='margin-top: 600px;'>
  <h4 >Finally, he mentioned meow!</h4>
</div>

</section>

::::

## Wrap it in a service

```
POST _analyze
{
  "analyzer": "standard",
  "text": "A KITTEN is-a juvenile cat.
  After being born, Kittens are totAlly dependent"
}
[ kitten, is, a, juvenile, cat,
after, being, born, kittens, are, totally,
dependent ]
```

:::

## Elasticsearch analyzers

* stopwords
* language
* pattern
* ...

:::

## analyzer componenents

* char filters
  * HTML strip
  * pattern replace
* tokenizers
  * word-oriented (letter, lowecase, whitespace)
  * partial words (n-grams, edge n-grams)
  * structured (pattern, char group)
* token filters
  * lowecase
  * ngram (min/max)
  * synonym, phonetic
  * ...

:::

### Analyzers

```
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_english_analyzer": {
          "type": "standard",
          "max_token_length": 5,
          "stopwords": "_english_"
        }
      }
    }
  }
}
```

:::

```
POST my_index/_analyze
{
  "analyzer": "my_english_analyzer",
  "text": "A KITTEN is-a juvenile cat.
  After being born, Kittens are totAlly dependent"
}
[ kitten, is, juven, ile, cat,
after, being, born, kitte, ns, are, total, ly,
depen, dent ]
```

::::

## Score

* TF-IDF
  * TF - _term frequency_ - wystąpienia tokenu w dokumencie
  * IDF - _inverted document frequency_ - iloraz ilości dokumentów przez ilość dokuemntów zawierających token
* _query normalization_
* _field normalization_
* metryka wektorowa pomiędzy zapytaniem i dokumentem (np. iloczyn skalarny)

:::
```
"kitten cat"
```
```
  "A kitten is a juvenile cat.
  After being born, kittens are totally dependent"

  "A dog is an uneducated creature."

  "Vindication is a part of every business."

  "Kites are great toys for cats."
```



::::

## Nieoczywiste zastosowania

::::

```
def ngrams(token, n, m)
  result = []
  for i in 0..token.length
    for j in n..m
      result << token[i,j]
    end
  end
  result.uniq
end

def analyze(text)
  normalize(tokenize(text).flat_map do |t|
    ngrams(t, 2, 5)
  end
end
```

:::

```
  text = "A kitten is a juvenile cat.
  After being born, kittens are totally dependent"
  analyze(text)
  #=> ["ki","kit", "kitt", "kitte", "it", "itt",
  #    "itte", "itten", "tt", "tte", "tten",
  #    "te", "ten", "en", "n", "",
  #    "ju", "juv", "juve",
  #    "juven", ...
```


:::

<img src='img/autocompletion.png' height=400/>

:::

```
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "tokenizer": "my_tokenizer"
        }
      },
      "tokenizer": {
        "my_tokenizer": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 5,
          "token_chars": [
            "letter",
            "digit"
          ]
        }
      }
    }
  }
}
```

:::

```
POST my_index/_analyze
{
  "analyzer": "my_analyzer",
  "text": "A kitten is a juvenile cat."
}
```
```
[ki, kit, kitt, kitte, is, ju, juv, juve, cat]
```

:::

### Completion

* prefix search `kitt.*`
  * poor scalability
* edge n-grams
  * fast search, slow indexing
* completion suggester
  * Finite State Transducer
  * no sorting, starts with the beginning
  * fast, but costly search
* other
  * prefix tree (trie), suffix trie
  * prefix hash tree

::::

### Not mentioned

* index implementation
* sharding & scalability
* distributed architecture
* scoring details

::::

## Summary

:::

## The magic happens in analyzers during indexing

:::

## I'm kidding


:::

## There is no magic.
## just software

:::

## Focus on proper indexes to make your searches simple and fast


::::

## References
* https://hackernoon.com/elasticsearch-building-autocomplete-functionality-494fcf81a7cf
* https://www.elastic.co/guide/en/elasticsearch/reference/current/search-analyzer.html
* https://www.elastic.co/guide/en/elasticsearch/reference/6.6/search-suggesters-completion.html
* https://www.elastic.co/guide/en/elasticsearch/guide/master/practical-scoring-function.html
* https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html
* https://medium.com/@prefixyteam/how-we-built-prefixy-a-scalable-prefix-search-service-for-powering-autocomplete-c20f98e2eff1
