#### ElasticSearch是什么

ES是一个高扩展、开源的全文索引和分析引擎，可以用于准时性的快速存储、搜索、分析等

#### 全文索引

全文索引是计算机索引程序通过扫描文章中的每一个词，**对每一个词建立一个索引**，指明该词在文章中出现的次数和位置，当用户查询的时候，检索程序就根据事先创建的索引进行查找，并根据查找到的结果反馈给用户。

类似于我们平时用的字典，通过字典定位到具体多少页，然后去找到该内容。为什么要记录页数还要记录出现次数呢，因为要体现权重

#### 为什么不适用数据库做索引

数据库不适合于数据量比较大的场景，数据量比较大的场景下，模糊匹配的性能极差，以及无法满足我们对业务的要求。还有就是数据库对大小写敏感，而所有引擎对大小写不敏感

#### 什么是Lucene

Lucene是JAVA的一个库，用于创建索引和检索，在没有SOLR和ES之前，大量使用到Luence作为企业级开发的中间件，此中间件可以把数据落盘以及检索。目前用到比较流行的ES和Solr也全是基于Luence开发的产物。但是Luence不是服务器，且操作难度比较大，所以目前大部分开发使用ES和Solr作为企业级开发的搜索引擎

#### ElasticSearch的应用场景

普遍用于站内搜索，什么是站内搜索，即使用目前站内已有的数据进行索引和检索，具体的使用场景如淘宝、京东等

#### ElasticSearch安装

到官网下载相应的包，进行解压安装

#### ElasticSearch的启动

    ./bin/elasticsearch 即可启动
    ./bin/elasticsearch -d 后台启动

然后访问http://localhost:9200即可访问，但是此访问方式比较有限，仅能本机访问，如果需要局域网访问或者公网访问，需要开启远程访问权限

**ES开启远程连接权限**

修改elasticsearch.yml文件的network配置，启动的时候出现一大堆错误问题，需要修改文件描述等等，主要是三个错误，具体自己去百度解决


#### ES相关的概念

ES近实时搜索，所谓近实时搜索就是不能完全做到索引完以后就能查询的到，但是相应的查询速度也是超级快的，基本上接近实时。基本上所有的操作不会超过1秒

- 索引

拥有几分相似特征的文档集合，必须是小写字母组成，类似于Mysql数据库的概念

- 类型

是索引的一个逻辑上的分类，类似于表的概念。ES6.X版本后一个索引中不能创建多个类型。这个时候还能对ES5.x多个类型有兼容。但是ES7-8中创建多个索引类型的API被删除

- 映射

类似于传统数据库中表的schema，用于定义索引中的类型的数据结构，用于限定类型的数据结构(包含了哪些字段、类型等等)

- 文档

类似于传统数据库中的一条记录，一个文档是一个可以被索引的基础信息单元，json格式

#### Kibana

1.什么是Kibana

kibana是estack组件，可以做es服务进行数据分析和操作，可以似为es的一个客户端

2.启动kibana

修改kibana.yaml的server.host和elasticsearch.url两个配置

    ./bin/kibana

访问http://localhost:6501

#### 索引操作

创建索引

    PUT /idx-name // 创建索引，索引名必须由小写字母组成

删除索引

    DELETE /idx-name // 删除索引

查看所有索引

    GET /_cat/indices   // 查看所有索引

查询所有索引的详细信息

    GET /_cat/indices?v  // 查看所有索引的详细信息

查看单个索引的信息

    GET /idx-name // 查看单个索引信息

删除所有索引

    DELETE /*  // 删除所有索引

查看索引和类型

    GET /idx-name/_mapping  // 查看索引和类型


#### 创建类型和映射

创建映射，目前版本一个索引只有一个类型，之所以把type放到mappings里面就是为了兼容之前版本

    PUT /idx-name

    {
      "mappings": {
        "properties": {
          "id": {
            "type": "integer"
          },
          "name": {
            "type": "keyword"
          },
          "age": {
            "type": "integer"
          },
          "date": {
            "type": "date"
          },
          "content": {
            "type": "text",
            "analyzer": "ik_max_word"
          }
        }
      }
    }

**mapping type的类型**

text, keyword, date, integer, long, double, boolean or ip

text和keyword类型的区别

- text

为文本类型，用于文本内容比较大的场景，需要分词，如个人介绍、商品介绍等

- keyworkd

为关键字类型，用于字符串比较少的地方，不需要分词，如邮件、姓名等


#### 文档操作

**添加文档**

    put /idx-name/_doc/id
    
    {
        "id": 1,
        "name": "asheng",
        "age": 12,
        "date": x
    }

**注意：**
这里的id和_id是一致的，只是_id是被隐藏了，一般建议使用_id代替id


**查询特定文档**

查询特定的一个文档，类似于getById

    // 查询某一个文档
    GET /idx-name/type/id


**更新文档**

此操作为部分更新，即仅更新一到多个字段，而非全部字段，如果不带/_update的时候，意味着会将原来文档删除，重新索引

    // 更新文档，在原始文档上更新
    POST /idx-name/_update/id
    {
        "doc":{
            "name":"asheng1"
        }
    }
    
    // 脚本更新
    {
        "script": "ctx._source.age += 1"
    }

**删除文档**

    // 删除文档
    DELETE /idx-name/_doc/1

**批量操作**

    批量更新文档:
    
    POST /_bulk?pretty
    
    # 索引操作
    { "index" : { "_index" : "idx-name", "_id" : "1" } }
    { "field1" : "value1" }
    // 删除操作
    { "delete" : { "_index" : "idx-name", "_id" : "2" } }
    // 添加操作
    { "create" : { "_index" : "idx-name", "_id" : "3" } }
    { "field1" : "value3" }
    // 更新操作
    { "update" : {"_id" : "1", "_index" : "idx-name"} }
    { "doc" : {"field2" : "value2"} }


#### 检索操作

ES提供了两种检索方式，第一种通过URL参数进行搜索，这种方法使用请求参数进行查询，类似于solr的用法，第二种是通过DSL(domain specified language)进行搜索。官方更推荐第二种方式基于JSON数据传递的方式进行交互，这种方式功能更加强大也更加简洁

1.根据id查询

    GET /idx-name/_doc/${_id}

2.查询所有

    GET /idx-name/_search?pretty
    {
      "query": {
        "match_all": {}
      }
    }

3.指定查询条数

    GET /idx-name/_search?pretty
    {
      "query": {
        "match_all": {}
      },
      "size":1
    }

4.分页查询

    GET /idx-name/_search?pretty
    {
      "query": {
        "match_all": {}
      },
      "size":1,
      "from": 2
    }

5.排序查询

    GET /idx-name/_search?pretty
    {
      "query": {
        "match_all": {}
      },
      "size":1,
      "from": 0,
      "sort": [
        {
          "age": {
            "order": "desc"
          }
        }
      ]
    }

6.指定查询字段

    GET /ems/_search?pretty
    {
      "query":{
        "match_all": {}
      },
      # 指定查询字段
      "_source":["feild1", "feild2"]
    }


7.关键字查询

    GET /ems/_search?pretty
    {
      "query": {
        "term": {
          #使用age进行精准匹配
          "age": {
            "value": 24
          }
          #使用content进行分词匹配
          "content":{
              "value": "spring"
          }
        }
      }
    }

Note:这里的查询即对**你输入的关键字**进行查询，term查询keyword类型，不会分词，需要完全匹配才行；term查询text类型，因为text字段会分词，而term不分词，所以term查询的条件必须是text字段分词后的某一个

8.范围查找

    GET /ems/_search?pretty
    {
      "query": {
        "range": {
          "age": {
            "gte": 10,
            "lte": 50
          }
        }
      }
    }

9.前缀查询

    GET /ems/_search?pretty
    {
      "query": {
        "prefix": {
          "content": {
            "value": "win"
          }
        }
      }
    }

前缀查询是区分大小写的，但是对于单词而言是有大小写的，而保存到es的时候，es会把单词处理成为小写进行保存的。是可以用于text的查询，查询text中任意出现此关键字的地方

10.通配符查询

    GET /ems/_search?pretty
    {
      "query": {
        "wildcard": {
          "content": {
            "value": "spr*g"
          }
        }
      }
    }

11.多id查询

    GET /ems/_search?pretty
    {
      "query": {
        "ids": {
          "values": ["eLV4_nUBJs_crxMhxkBZ", "ebV4_nUBJs_crxMhxkBZ"]
        }
      }
    }

12.模糊查询

    GET /ems/_search?pretty
    {
      "query": {
        "fuzzy": {
          "content": "sping1"
        }
      }
    }

模糊查询仅能处理错误字符在0-2个之间，一旦超过0-2以后就匹配不上了。如果**关键字**小于等于2的时候，必须完全匹配，如果3-5个字允许1个错误。如果大于5个允许2个错误。大于2个错误的时候就不行了。


13.bool查询

- must:必须满足

- should: 满足一个就行

- must_not: 刚好must相反，必须不满足才行


    GET /ems/_search?pretty
    {
      "query": {
        "bool": {
          "must": [
            {
              "term": {
                "age": {
                  "value": "24"
                }
              }
            }
          ],
          "must_not": [
            {
              "ids": {"values": ["RrV3_nUBJs_crxMhD0CV"]}
            }
          ]
        }
      }
    }

14.多字段查询

    GET /ems/_search?pretty
    {
      "query": {
        "multi_match": {
          "query": "小黑",
          "fields": ["name", "content"]
        }
      }
    }

这里不仅仅会对多字段进行查询，也会对查询的关键字进行分词然后再查询。即会对query的内容进行进行分词后，然后用分词后的内容进行查询。这时如果query的内容和多字段命中的越多，则得分越高。如果匹配的字段分词就会进行分词查询，如果不分词就会拿着整体进行查询

15.多字段分词查询

    GET /ems/_search?pretty
    {
      "query": {
        "query_string": {
         "default_field": "content",
          "query": "这个是张小五",
          "fields": ["name", "content"],
          "analyzer": "standard"
        }
      }
    }

这个和上面的多字段查询比较类似，其中default_field和fields只能使用一个，但是区别在于可以自定义analyzer分词器

16.高亮查询

    GET /ems/_search?pretty
    {
      "query": {
        "term": {
          "content": {
            "value": "spring"
          }
        }
      },
      // 高亮配置
      "highlight": {
        // 高亮字段
        "fields": {
          "content": {}
        },
        // 高亮的自定义标签
        "pre_tags": ["<span style='color=red'>"],
        "post_tags": ["</span>"],
        // true即query条件里满足的字段才能高亮
        "require_field_match":false
      }
    }

17.match查询

    GET /ems/_search
    {
      "query": {
        "match": {
          "content": "我是 开发"
          // 默认分词是以or条件处理的，可以替换为and要求必须同时满足
          "operator":"or"
        }
      }
    }
    
    GET /ems/_search
    {
      "query": {
        "match": {
          "content": {
            "query": "sproogboot",
            "fuzziness": "AUTO",
            "prefix_length": 3
          }
        }
      }
    }

match查询，当为keyword类型的时候，查询内容不会被分词，必须全部匹配才行；当为text类型的时候，查询内容会被分词，只要match的分词结果和text的分词结果有相同的就匹配。和term查询的区别是，term输入的查询条件不会被分词，而match查询输入的内容是会被分词的，其中分词后的结果以or条件进行查询，即命中其中一个分词即可。

match匹配也可以用于模糊查询，即和fuzzy查询类似，auto的规则和fuzzy的规则一直，可选参数为0、1、2、auto，而auto会根据输入的关键字长度决定，一般我们会直接使用默认的auto；而prefix_length长度是决定了前多少个字符必须正确的，否则匹配不上，因为我们平时的拼写都错在后面的字段上，因此要求前多少位必须正确

18.match_phrase查询

    GET /ems/_search
    {
      "query": {
        "match_phrase": {
          "content": "擅长于开发各种服务器插件等等"
        }
      }
    }

match_phrase查询，当keyword类型必须一直才行。为text类型的时候，输入内容会分词，分词结果必须在text字段分词中都包含，而且顺序必须相同，而且必须都是连续的。query_string查询类型和match_phrase一样，但是不需要连续

#### 分词器

什么叫分词，就是提炼出关键字。主要特点就是拆分关键词，去掉停用词和语气词

ES提供的分词器

- 默认的 标准分词器 standard analyzer

英文:单词分词 中文:单字分词

    GET _analyze
    {
      "analyzer": "standard",
      "text":"我是中国人"
    }

- 简单分词器 simple analyzer

英文:单词分词 去掉数字，中文:不分词，就整个句子

    GET _analyze
    {
      "analyzer": "simple",
      "text":"我是中国人"
    }

- IK分词器，必须和ES的版本一致

英文:单词分词

中文:词语分词，分为ik_smart和ik_max_word两种。主要区别是ik_max_word会进行消息拆分和组合

    GET _analyze
    {
      "analyzer": "ik_max_word",
      "text":"我是中国人"
    }

在线安装 ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.9.3/elasticsearch-analysis-ik-7.9.3.zip

Note:如果需要对字段分词，需要在构建索引的时候使用，类似如下，其他分词器的排行等等详细见地址:https://github.com/magese/ik-analyzer-solr

    {
        "mappings":{
            "ems":{
                "properties":{
                    "introduces":{
                        "type":"text",
                        "analyzer":"ik_max_word"
                    }
                }
            }
        }
    }

**自定义扩展词**

有些关键字需要根据业务来自定义，但是ik不能识别这些自定义关键字，所以这个时候就需要自定义扩展词，并且添加到ik中，以后就能自动识别

**配置IK的扩展词**

文件位置: ./elasticsearch/config/analysis-ik

其中IKAnalyzer.cfg.xml中有扩展词和停用词的用法说明，在使用文件格式的时候，文件必须以.dic扩展名结尾

**自定义停用词**

有些关键词因为业务影响不能作为关键字，比如某些大佬的名字等，不能让其搜索到，就可以添加到停用词里。具体的操作和扩展词配置类似

**配置远程词典**

当机器部署的比较多的时候，会出现一个问题，就是每台机器上都要配置一样的，以及需要同步到多台机器上并且机器要重新启动，这种方式不合理，因此需要远程词典。其实就是一个uri，定位到一个txt文件即可，不过需要文件格式为UTF-8

#### 过滤查询

过滤查询的效率比较高，仅查询符合条件的数据，不会计算文档的得分和排序。当组合使用的时候，会先执行filter然后再执行其他的查询

**基本使用**

    GET /ems/_search
    {
      "query": {
        "bool": {
          "must": [
            {
              "term": {
                "name": {
                  "value": "盛问问"
                }
              }
            }
          ], 
          "filter": [
            {
              "range": {
                "age": {
                  "gte": 25
                }
              }
            }
          ]
        }
      }
    }


#### 重新索引

根据以上内容，我们发现，如果我们的索引格式发生了改变，或者是我们增加了新的停用词和扩展词的时候，我们之前建立的索引就会和目前的数据不一致。

- 索引格式方式改变，比如刚开始定义为keyword后面需要为text类型，这个是不被允许的。需要删除索引才行，但是我们不能删除

- 增加了新的扩展词，比如盛义文刚开始不是扩展词，会被识别为盛、义、文。仅当添加为扩展词时才会被识别，但是老的文档是没被分词的

基于上述的情况，我们需要对老的文档进行重新索引，而重新索引的方式为创建一个新的索引，将老的索引内容重新索引到新的索引里，然后将新的索引重命名，最后删除老的索引

**重新索引的方式**

    POST /_reindex
    {
        "source":{
            "index":"old-idx"
        },
        "dest":{
            "index":"new-idx"
        }
    }

上面的方式将会以同步执行的方式进行索引，如果索引内容较大的话，建议在url后加上**?wait_for_completion=false**参数

**替换别名**

    POST /_aliases
    {
        "action":[{
            "add": {
                "index": "new-idx",
                "alias": "idx",
            },
            "remove": {
                "inex":"old-idx",
                "alias":"idx-alias"
            }
        }
        ]
    }


#### ES集群

ES本身就是分布式应用，即使是单节点部署也是集群模式。ES的集群模式主要是应对高并发和高可用，以及降低索引对磁盘的压力。在集群中主要有两个概念，分别是shard和replica，创建索引的时候默认是5个shard和2个replica。es对外服务是透明的，即无论是访问整体集群还是单台机器的方式是一致的，当进行查询的时候，会把各个服务的数据检索起来然后聚合后返回给客户，具体的做法开发人员不用关注

- shard

shard即分片，主要是应对高并发，在数据量比较大的时候，会对多台机器上进行分片，让数据均匀的分布在各台机器上，可以减轻每台机器的压力以及能够存储更多数据

- replica

replica即复制，主要是应对高可用，在有些机器宕机的情况下依旧能够提供完整的服务，其中集群内的机器相互备份，不会把索引在本台机器上的数据备份到本台机器上。即使这样，当机器宕机过多的时候，服务仍然不可用。索引要查看相互备份的索引是否完整。为green的时候非常健康，为yellow的时候表示服务可用，但是有的机器已经宕机，为red的时候，表示服务已经不可用，已经不能提供完整的索引


参考文档：

- [官方文档链接](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)

- [Match匹配博客](https://blog.csdn.net/weixin_34128534/article/details/88690764)

- [Api文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.x/java-rest-high-search.html#_specifying_sorting)
