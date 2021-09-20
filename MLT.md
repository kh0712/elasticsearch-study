# More like this 쿼리



> The More Like This Query finds documents that are "like" a given set of documents. In order to do so, MLT selects a set of representative terms of these input documents, forms a query using these terms, executes the query and returns the results. The user controls the input documents, how the terms should be selected and how the query is formed.



More Like This (이하 MLT) 쿼리는 입력으로 주어진 도큐먼트 집합과 비슷한 도큐먼트들을 찾는다. 그렇게 하기 위해서, MLT 쿼리는 입력으로 들어온 도큐먼트를 대표하는 텀들을 고르고, 고른 텀들을 이용해 쿼리를 만들어 실행한다. 유저는 입력되는 도큐먼트와 텀들이 어떻게 선택될 지, 그리고 쿼리가 어떻게 형성될 지 조절할 수 있다.



> The simplest use case consists of asking for documents that are similar to a provided piece of text. Here, we are asking for all movies that have some text similar to "Once upon a time" in their "title" and in their "description" fields, limiting the number of selected terms to 12.



가장 단순한 사용 예시는 입력으로 부분 텍스트를 넣는 것이다. 영화 데이터가 엘라스틱서치에 인덱싱 되어 있을 때, 영화의 제목과 설명 필드에 "Once upon a time" 과 비슷한 내용을 가지고 있는 영화를 찾아보자. 그리고 쿼리를 형성할 때 선택되는 텀의 개수를 12개로 제한해보자.



```json
GET /_search
{
  "query": {
    "more_like_this" : {
      "fields" : ["title", "description"],
      "like" : "Once upon a time",
      "min_term_freq" : 1,
      "max_query_terms" : 12
    }
  }
}
```



>A more complicated use case consists of mixing texts with documents already existing in the index. In this case, the syntax to specify a document is similar to the one used in the [Multi GET API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-multi-get.html).



좀 더 복잡한 사용 예시로 입력값을 이미 인덱싱되어있는 도큐먼트와 임의의 문자열을 넣는 경우를 살펴보자. 이 경우, Multi GET API와 문법이 비슷하다.



```json
GET /_search
{
  "query": {
    "more_like_this": {
      "fields": [ "title", "description" ],
      "like": [
        {
          "_index": "imdb",
          "_id": "1"
        },
        {
          "_index": "imdb",
          "_id": "2"
        },
        "and potentially some more text here as well"
      ],
      "min_term_freq": 1,
      "max_query_terms": 12
    }
  }
}
```





>Finally, users can mix some texts, a chosen set of documents but also provide documents not necessarily present in the index. To provide documents not present in the index, the syntax is similar to [artificial documents](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-termvectors.html#docs-termvectors-artificial-doc).



마지막으로 임의의 문자열과 인덱싱 되어있지 않은 도큐먼트를 입력값으로 넣어보자. 인덱싱 되어있지 않은 도큐먼트를 입력값으로 넣기 위해 사용하는 문법은 Term vectors API의 artificial documents 문법과 비슷하다.



```json
GET /_search
{
  "query": {
    "more_like_this": {
      "fields": [ "name.first", "name.last" ],
      "like": [
        {
          "_index": "marvel",
          "doc": {
            "name": {
              "first": "Ben",
              "last": "Grimm"
            },
            "_doc": "You got no idea what I'd... what I'd give to be invisible."
          }
        },
        {
          "_index": "marvel",
          "_id": "2"
        }
      ],
      "min_term_freq": 1,
      "max_query_terms": 12
    }
  }
}
```



---





## More Like This 쿼리의 작동 원리



> Suppose we wanted to find all documents similar to a given input document. Obviously, the input document itself should be its best match for that type of query. And the reason would be mostly, according to [Lucene scoring formula](https://lucene.apache.org/core/4_9_0/core/org/apache/lucene/search/similarities/TFIDFSimilarity.html), due to the terms with the highest tf-idf. Therefore, the terms of the input document that have the highest tf-idf are good representatives of that document, and could be used within a disjunctive query (or `OR`) to retrieve similar documents. The MLT query simply extracts the text from the input document, analyzes it, usually using the same analyzer at the field, then selects the top K terms with highest tf-idf to form a disjunctive query of these terms.



입력값으로 넣는 도큐먼트와 가장 비슷한 도큐먼트를 찾는 상황을 가정해보자. 분명하게도, 입력 도큐먼트와 가장 비슷한 도큐먼트는 입력 도큐먼트 그 자체일 것이다. 그 이유는, 루씬 스코어 계산 공식에 따르면, 입력 도큐먼트의 텀들이 입력 도큐먼트와 비교했을 때 높은 tf-idf 점수를 얻기 때문이다. 그러므로 입력 도큐먼트의 텀들 중에서 높은 tf-idf 스코어를 기록한 텀들이 입력 도큐먼트를 가장 좋게 표현한다고 할 수 있다. 그리고 그 대표하는 텀들을 논리합 쿼리(OR 쿼리같은)로 조합해서 인덱스 내 비슷한 도큐먼트를 찾을 수 있다. MLT 쿼리는 단순히 입력된 도큐먼트에서 텍스트를 추출하고, 추출된 텍스트는 애널라이저를 거쳐 - 보통 해당 필드 인덱싱에 사용된 애널라이저를 사용한다. - 텀들로 분리되며, 분리된 텀 중 인력된 도큐먼트와 tf-idf로 비교했을 때 높은 tf-idf 스코어를 가지는 K 개의 텀을 선별한다. 그리고 선별된 K개의 텀들은 논리합 쿼리에 이용되어 비슷한 도큐먼트를 찾는데 사용된다.



> Important
>
> The fields on which to perform MLT must be indexed and of type `text` or `keyword`
>
> Additionally, when using `like` with documents, either `_source` must be enabled or the fields must be `stored` or store `term_vector`. 
>
> In order to speed up analysis, it could help to store term vectors at index time.

중요: MLT 쿼리에 사용되는 필드들은 text 또는 keyword 타입이어야 한다. 추가적으로 like로 도큐먼트를 지정해서 쓸 때 원본 전체(_source)가 저장되어 있어야 하거나 필드들이 저장되어 있거나(store 옵션 true) 텀벡터가 저장되어 있어야 한다. 빠른 실행속도를 위해서는 인덱싱 타임에 term vectors를 저장하는 것이 도움이 될 수 있다.



> For example, if we wish to perform MLT on the "title" and "tags.raw" fields, we can explicitly store their `term_vector`at index time. We can still perform MLT on the "description" and "tags" fields, as `_source` is enabled by default, but there will be no speed up on analysis for these fields.



예를 들어 title과 tags.raw 필드에 대해 MLT 쿼리를 사용한다고 해보자. 도큐먼트를 인덱싱 할 때, 명시적으로 term_vector를 문서 내부에 저장할 수 있다. 또한 term_vector를 저장하지 않았더라도 원본이 저장된 description 필드와 tags 필드에 대해서도 MLT 쿼리를 실행할 수 있는데, 속도 측면의 성능 상의 이점은 없다.



```json
PUT /imdb
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "term_vector": "yes"
      },
      "description": {
        "type": "text"
      },
      "tags": {
        "type": "text",
        "fields": {
          "raw": {
            "type": "text",
            "analyzer": "keyword",
            "term_vector": "yes"
          }
        }
      }
    }
  }
}
```



---



## MLT 쿼리의 파라미터



>The only required parameter is `like`, all other parameters have sensible defaults. There are three types of parameters: one to specify the document input, the other one for term selection and for query formation.



유일한 필수 파라미터는 like 파라미터이다.  다른 파라미터들은 따로 지정하지 않아도 적절히 사용할 수 있는 기본 값을 가지고 있다. 세 종류의 파라미터가 있으며, 각각은 입력 도큐먼트를 명시하기 위한 파라미터, 텀 선택에 관한 파라미터, 쿼리 생성에 관한 파라미터이다.



이하 원문은 원본 문서를 참고

### 입력 도큐먼트와 관련된 파라미터



| 파라미터 | 설명                                                         |
| -------- | ------------------------------------------------------------ |
| like     | like 파라미터는 MLT 쿼리에서 유일한 필수 파라미터이며, 융통성 있는 문법을 따른다. 즉 자유로운 형식의 텍스트를 입력할 수도, 하나 혹은 여러 도큐먼트를 지정할 수도 있다. 도큐먼트를 지정하는 방법은 Multi GET API와 비슷하다. 도큐먼트를 지정할 때 텍스트는 각 도큐먼트 리퀘스트에 의해 오버라이드 되지 않는다면(중복되지 않는다면) 아래에 설명되어 있는 파라미터인  `fields`에서 가져온다. 텍스트는 해당 필드의 애널라이저에 의해 분석되나, 오버라이드 될 수 있다. 애널라이저를 오버라이드 하는 방법은 Term Vectors API의 per_field_analyzer를 지정하는 문법과 비슷하다. 추가적으로 입력값으로 인덱스에 존재하지 않는 도큐먼트를 넣으려면 artificial documents 문법을 사용하면 된다. |
| unlike   | unlike 파라미터는 like와 결합하여 선택되지 않을 텀들을 지정하는데 사용할 수 있다. 다시 말해서, `like:"Apple"`, `unlike: "cake"` 처럼 쓸 수 있다. 문법은 like와 동일하다. |
| fields   | 분석에 사용할 텍스트 필드들의 목록을 지정한다. 기본적으로 메타데이터를 제외한, term-level 쿼리에 사용될 수 있는 필드들이 포함된다. |



### 텀 선택과 관련된 파라미터

| 파라미터        | 설명                                                         |
| --------------- | ------------------------------------------------------------ |
| max_query_terms | 최대 몇개의 텀이 선택될 지 정하는 파라미터. 개수를 올리면 정확도는 올라가나 쿼리 성능은 떨어진다. 기본값은 25 |
| min_term_freq   | 텀과 문서의 tf를 계산했을 때, tf가 몇 이하면 무시할 지 정하는 파라미터. 기본은 2 |
| min_doc_freq    | 텀이 몇 개 이하의 도큐먼트에서 나왔을 때 무시할 지 정하는 파라미터. 기본은 5 |
| max_doc_freq    | 텀이 몇개 이상의 도큐먼트에서 나왔을 때 무시할 지 정하는 파라미터. 모든 문서에서 등장하는 텀은 별 가치가 없다. 기본은 Integer.MAX_VALUE |
| min_word_length | 텀의 최소 길이. 기본은 0                                     |
| max_word_length | 텀의 최대 길이. 기본은 제한 없음(0)                          |
| stop_words      | 불용어 리스트를 지정한다.                                    |
| analyzer        | 자유 형식의 텍스트를 어떤 애널라이저로 분석할 지 지정한다. 기본은 `fields` 의 첫번째 필드의 애널라이저이다. |



### 쿼리 형성에 관련된 파라미터

| 파라미터                  | 설명                                                         |
| ------------------------- | ------------------------------------------------------------ |
| minimum_should_match      | 논리합 쿼리를 만들 때 몇개의 텀이 맞아야 하는지 결정한다. 문법은 minimum should match와 동일하며 기본은 30퍼센트이다. |
| fail_on_unsupported_field | 지원되지 않는 타입이 있을 때(text나 keyword가 아닌 타입) 쿼리를 실패시킬 지 혹은 무시할지 결정하는 파라미터 기본은 true이다. |
| boost_terms               | 각 term에 대한 tf-idf 계산용 부스트 파라미터 기본은 사용 안함(0)이다. |
| include                   | 인풋 도큐먼트가 결과에 포함될지 안될지 결정하는 파라미터. 기본은 false이다. |
| boost                     | 전체 쿼리에 대한 부스트 파라미터. 기본은 1이다.              |



---



## MLT 대신 사용할 수 있는 것들



> To take more control over the construction of a query for similar documents it is worth considering writing custom client code to assemble selected terms from an example document into a Boolean query with the desired settings. The logic in `more_like_this` that selects "interesting" words from a piece of text is also accessible via the [TermVectors API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-termvectors.html). For example, using the termvectors API it would be possible to present users with a selection of topical keywords found in a document’s text, allowing them to select words of interest to drill down on, rather than using the more "black-box" approach of matching used by `more_like_this`.



쿼리를 생성할 때 더 세밀하게 조절하기 위해 불린 쿼리를 사용하는 것이 도움이 된다. ""흥미로운" 단어를 선택하는 MLT의 로직은 TermVectors API를 이용해 확인할 수 있다. 세부 구현을 알기 어려운 블랙박스 식의 MLT 쿼리를 사용하는 것 대신에 termvectors API를 활용해서 도큐먼트를 대표하는 키워드들을 찾을 수 있고, 흥미로운 키워드를 선택하는 과정을 구체화할 수 있다.