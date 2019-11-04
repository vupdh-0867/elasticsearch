# elasticsearch

## 1 Giới thiệu

- Là một open-source search engine được xây dựng dựa trên apache lucene (một thư viện mã nguồn mở để xây dựng các search engine), nói cách khác lucene giống như ruby còn es là rails

- ES hoạt động độc lập như một server và giao tiếp thông qua giao thức resful

- ES là một hệ phân tán *

2 Các khái niệm cơ bản

- Document: là một JSON object, là đơn vị nhỏ nhất để lưu trữ dữ liệu của ES

- Shard: gồm primary shard và replica shard, các document sẽ được lưu trong primary shard và được duplicate trong replica shard cho mục đích khôi phục trong trường hợp có 2 node trở lên

- Index: khi dữ liệu được gửi lên ES thì nó sẽ được đánh index trước khi được lưu vào shard, dữ liệu được đánh index dựa theo một cấu trúc là inverted index, ở SQL là Btree, dữ liệu sẽ được tách ra thành các từ gọi là term

## 2 Analysis data

Gồm bốn bước:

**Character filtering**: Chuyển đổi các ký tự sử dụng character filter
**Breaking text into tokens**: Tách đoạn text thành tập hợp các token
**Token filtering**: Biến đổi các token sử dụng token filter
**Token indexing**: Và cuối cùng là lưu các token đó vào inverted index

![inverted index](./images/inverted_index.png)

Ví dụ: Sơ đồ phân tích đoạn text "share your experience with NoSql & big data technologies"

![eg analysis](./images/eg_analysis.png)

### 2.1 Character filtering

Đây là bước đầu tiên của quá trình analysis, ở bước này, các ký tự sẽ được chuyển đổi thành dữ liệu cho phù hợp với yêu cầu search của bạn sử dụng các Character filters , quá trình nãy sẽ giúp bạn xử lý cho các trường hợp như muốn loại bỏ các thẻ/ký tự của HTML trong đoạn text, chuyển các ký tự thành từ có nghĩa như “I love u 2” thành “I love you too”. Ở ví dụ trên charater filter đã chuyển đổi ký tự "&" thành từ "and", vì vậy khi bạn tìm kiếm với từ khóa "and" thì dữ liệu chứa ký tự "&" sẽ được liệt kê ra

### 2.2 Breaking into tokens

Sau khi đoạn text đã được xử lý chuyển đổi các ký tự xong, nó sẽ được phân tách thành các tokens độc lập sử dụng các tokenizers. Elasitcsearch cung cấp rất nhiều tokenizers để phục vụ cho yêu cầu bài toán của bạn, ví dụ như whitespace tokenizer sẽ tách đoạn text thành các tokens dựa váo các khoảng trắng whitespace: "artic region" sẽ output ra 2 token artic, region, hoặc letter tokenizer sẽ tách đoạn text thành các token dựa vào whitespace và các ký tự đặc biệt: "sun-asterisk company" sẽ có output là 3 tokens sun, asterisk, company

### 2.3 Token filtering

Sau khi đoạn text được tách và cho ra output là các tokens, các tokens này sau đó sẽ được đưa vào một hoặc nhiều các Token filters, tại đây các tokens sẽ được xóa bợt, thêm hoặc chỉnh sửa tùy vào loại token filter. Các token này sẽ hữu ích trong trường hợp bạn muốn chuyển các token về dạng lowercase và ngược lại, hoặc có thể thêm token mới "tools" như ở ví dụ trên. Một analyzer có thể có không hoặc nhiều token filters

### 2.4 Token indexing

Sau khi các token đã đi qua 0 hoặc nhiều token filters chúng đã được gửi tới Lucene để được lập đánh index. Một analyzer sẽ bao gồm không hoặc nhiều character filters, một tokenizer, và không hoặc nhiều token filters

*Tạo một index*

```json
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
    "settings" : {
        "index" : {
            "number_of_shards" : 1,
            "number_of_replicas" : 1
        }
    }
}
'
```

*Cấu trúc của một Analyzer*

```json
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom",
          "tokenizer": "whitespace",
          "char_filter": [
            "my_custom_char_filter"
          ],
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      },
      "char_filter": {
        "my_custom_char_filter": {
          "type": "mapping",
          "mappings": [
            "& => and"
          ]
        }
      }
    }
  }
}
'
curl -X POST "localhost:9200/my_index/_analyze?pretty" -H 'Content-Type: application/json' -d'
{
  "analyzer": "my_custom_analyzer",
  "text": "share <b>your</b> experience with NoSql & big data technologies"
}
'
```

*Custom các charfilter, tokenizer, token filter*

```json
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom",
          "char_filter": [
            "emoticons"
          ],
          "tokenizer": "punctuation",
          "filter": [
            "lowercase",
            "english_stop"
          ]
        }
      },
      "tokenizer": {
        "punctuation": {
          "type": "pattern",
          "pattern": "[ .,!?]"
        }
      },
      "char_filter": {
        "emoticons": {
          "type": "mapping",
          "mappings": [
            ":) => _happy_",
            ":( => _sad_"
          ]
        }
      },
      "filter": {
        "english_stop": {
          "type": "stop",
          "stopwords": "_english_"
        }
      }
    }
  }
}
'
curl -X POST "localhost:9200/my_index/_analyze?pretty" -H 'Content-Type: application/json' -d'
{
  "analyzer": "my_custom_analyzer",
  "text": "I\u0027m a :) person, and you?"
}
'
```

List tất cả các analyzer trong index
```json
curl -X GET "localhost:9200/my_index/_settings?pretty"
```

Update analyzer cho một index có sẵn
```json
curl -X POST "localhost:9200/my_index/_close?pretty"
curl -X PUT "localhost:9200/my_index/_settings?pretty" -H 'Content-Type: application/json' -d'
{
  "analysis" : {
    "analyzer":{
      "aaaa":{
        "type":"custom",
        "tokenizer":"whitespace"
      }
    }
  }
}
'
curl -X POST "localhost:9200/my_index/_open?pretty"
```

### 2.5 Custom một char filter
