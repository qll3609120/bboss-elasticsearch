# 通过Function Score Query优化Elasticsearch搜索结果(综合排序)

在使用Elasticsearch进行全文搜索时，搜索结果默认会以文档的相关度进行排序，如果想要改变默认的排序规则，也可以通过**sort**指定一个或多个排序字段。但是使用sort排序过于绝对，它会直接忽略掉文档本身的相关度（根本不会去计算）。在很多业务场景下这样做的效果并不好，这时候就需要对多个字段进行综合评估，得出一个最终的排序。

本文涉及到的程序和配置文件对应的完整可运行的java工程源码地址：

https://github.com/rookieygl/bboss-wiki

# 1.function\_score介绍

在Elasticsearch中function_score是用于处理文档分值的 DSL，它会在查询结束后对每一个匹配的文档进行一系列的重打分操作，最后以生成的最终分数进行排序。它提供了几种默认的计算分值的函数：

- weight：设置权重
- field_value_factor：将某个字段的值进行计算得出分数。
- random_score：随机得到 0 到 1 分数
- 衰减函数（decay functions）：同样以某个字段的值为标准，距离某个值越近得分越高
- script_score：通过自定义脚本计算分值，可引入脚本，传入参数等

function_score通过属性`boost_mode`可以指定计算后的分数与原始的`_score`如何合并，有以下选项：

- multiply：将结果乘以_score

- sum：将结果加上_score

- min：取结果与_score的较小值

- max：取结果与_score的较大值

- replace：使结果替换掉_score

function_score通过属性score_mode每一个函数都会给文档一个评分，因此我们需要把这些函数返回的分数归约成一个总分数，然后和_score组合成一个新的最终分数。该参数score_mode指定函数得分的组合方式：

- multiply: 函数结果会相乘(默认行为)
- sum：函数结果会累加

- avg：得到所有函数结果的平均值

- max：得到最大的函数结果

- min：得到最小的函数结果

- first：只使用第一个函数的结果，该函数可以有过滤器，也可以没有

function_score通过设置max_boost参数，可以将分数限制为不超过某个值，限制该函数的最大影响 力，当然max_boost只是对函数的结果有所限制，并不是最终的_score。默认max_boost值为FLT_MAX（结果的最大浮点数，即为无限制）。

min_score：默认情况下，修改分数不影响文档的匹配。要排除不符合指定分数的文档，function_score通过min_score可以将参数设置为所需最低分数；当然要对查询返回的所有文档进行评分，然后逐个判断过滤。

**注**

​	**max_boost是对每个函数都进行限制。**



function_score的作用就是综合各个函数的得分，因此注意两点：

1. **function_score如果没有给函数设置过滤器或者query条件，这相当于指定 `"match_all": {}`。**

2. **每个函数产生的分数不决定排名，因为我们只要最终得分，总分数越高，排名越靠前。**
3.  **sort排序不参与评分，导致function_score无效，谨慎结合使用。**

接下来本文将举例详细介绍这些函数的用法，以及它们的使用场景。

# 2.function_score使用案例

## 2.1 案例准备工作

本文以一个商品检索作为案例来介绍function_score的具体用法。

在开始之前先在工程中创建Bboss的DSL配置文件，本文中涉及的配置都会加到里面：resources/esmapper/function_score.xml

https://github.com/rookieygl/bboss-wiki/blob/master/src/main/resources/esmapper/function_score.xml

Java测试类：com/bboss/hellword/FunctionScore/FunctionScoreTest

https://github.com/rookieygl/bboss-wiki/blob/master/src/test/java/com/bboss/hellword/FunctionScore/FunctionScoreTest.java

### 2.1.1 定义索引-商品索引mapping定义和配置

在配置文件中添加商品索引mapping定义-createItemsIndice

```xml
     <!--
     通过function_score函数计算相关度打分案例
     参考官方文档
     https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html
    -->
    <!--
    创建商品索引items mappings dsl
    -->
<property name="createItemsIndice">
        <![CDATA[{
            "settings": {
                "number_of_shards": 1,
                "number_of_replicas": 0,
                "index.refresh_interval": "5s"
            },
            "mappings" : {
                "item" : {
                    "properties" : {
                        "docId" : {
                            "type" : "long"
                        },
                        "name" : {
                            "type" : "text",
                            "fields" : {
                              "keyword" : {
                                "type" : "keyword",
                                "ignore_above" : 256
                              }
                            }
                        },
                        "sales" : {
                            "type" : "long"
                        },
                        "title" : {
                            "type" : "text",
                            "fields" : {
                              "keyword" : {
                                "type" : "keyword",
                                "ignore_above" : 256
                              }
                            }
                        }
                    }
                }
            }
        }]]>
</property>
```

### 2.1.2 创建索引-加载配置文件并创建索引

```java
/**
* 创建商品索引
*/
@Test
public void dropAndCreateItemsIndice() {
        ClientInterface clientInterface = ElasticSearchHelper.getConfigRestClientUtil("esmapper/functionscore.xml");
        if (clientInterface.existIndice("items")) {
            clientInterface.dropIndice("items");
        }
        clientInterface.createIndiceMapping("items", "createItemsIndice");
}
```

### 2.1.3 准备测试数据-批量添加商品数据

通过以下代码向item索引中添加不同的测试数据：

```java
/**
* 导入商品数据
*/
@Test
public void insertItemsData() {
        ClientInterface clientInterface =bbossESStarter.getConfigRestClient("esmapper/functionscore.xml");
        List<Item> items = new ArrayList<>();
        Item item1 = new Item();
        Item item2 = new Item();
        Item item3 = new Item();
        Item item4 = new Item();

        item1.setDocId(1L);
        item1.setTitle("雨伞");
        item1.setName("天堂伞");
        item1.setSales(500L);

        item2.setDocId(2L);
        item2.setTitle("雨伞");
        item2.setName("宜家");
        item2.setSales(1000L);

        item3.setDocId(3L);
        item3.setTitle("巧克力");
        item3.setName("德芙");
        item3.setSales(100000L);

        item4.setDocId(4L);
        item4.setTitle("奶糖");
        item4.setName("大白兔");
        item4.setSales(1000000000L);


        items.add(item1);
        items.add(item2);
        items.add(item3);
        items.add(item4);
        //强制refresh，以便能够实时执行后面的检索操作，生产环境去掉"refresh=true"
        String response = clientInterface.addDocuments("items", "item", items, "refresh=true");
        logger.debug(response);

}
```

以上创建索引和添加数据的代码的类似代码不再重复贴出，也可以用kibana添加数据。

## 2.2 weight

weight 的用法最为简单，只需要设置一个数字作为权重，文档的分数就会乘以该权重。
他最大的用途应该就是和过滤器一起使用了，因为过滤器只会筛选出符合标准的文档，而不会去详细的计算每个文档的具体得分，所以只要满足条件的文档的分数都是 1，而 weight 可以将其更换为你想要的数值。

## 2.3 field\_value\_factor

field\_value\_factor 的目的是通过文档中某个字段的值计算出一个分数，它有以下属性：

- field：指定字段名

- factor：对字段值进行预处理，乘以指定的数值（默认为 1），可以用来平衡评分权重

- modifier将字段值进行加工，有以下的几个选项：

  none：不处理(默认值)

  log：计算对数

  log1p：先将字段值 +1，再计算对数

  log2p：先将字段值 +2，再计算对数

  ln：计算自然对数

  ln1p：先将字段值 +1，再计算自然对数

  ln2p：先将字段值 +2，再计算自然对数

  square：计算平方

  sqrt：计算平方根

  reciprocal：计算倒数

**注**

​	**field\_value\_factor 只能对数字类型字段评分,且一个函数只能指定一个字段**

举一个简单的例子，假设有一个商品索引，搜索时希望在相关度排序的基础上，销量（sales）更高的商品能排在靠前的位置，那么这条查询 DSL 可以是这样的：在esmapper/functionscore.xml定义一个名称为testFieldValueFactor的按照商品销量（sales）排序的DSL

```xml
<!--指定sales字段排序-->
<property name="testFieldValueFactor">
        <![CDATA[
        {
			## 配合过滤器使用，文档与查询次相关字数多的的
			"explain": true,
            "query": {
                "function_score": {
                    "query": {
                        "match": {
                            "title": #[titleName]
                        }
                    },
                    "field_value_factor": {
						## 指定需要打分的字段
                        "field": #[valueFactorName],
						## 排序评分函数
                        "modifier": "log1p",
                        "factor": 0.1
                    },
					
					## 与文档评分的组合方式
                    "boost_mode": "sum"
                }
            },
            "from": #[from],
            "size": #[size]
        }
        ]]>
</property>
```

这条查询会将品类为雨伞的商品检索出来，然后对这些文档计算一个与销量相关的分数，并与之前查询分数相加，对应的公式为：
$$
Score = Score + log (1 + 0.1 * sales)
$$
factor为打分系数，和sales相乘，modifier为评分函数，boost_mode决定了评分函数和查询分数的组合方式

执行上面的dsl：

```java
/**
*指定sales字段排序
*/
@Test
public void testFieldValueFactor(){
        ClientInterface clientUtil = bbossESStarter.getConfigRestClient("esmapper/functionscore.xml");
        Map<String,Object> queryMap = new HashMap<>();
        // 指定商品类目作为过滤器
        queryMap.put("titleName","雨伞");
        // 指定需要field_value_factor运算的参数
        queryMap.put("valueFactorName","sales");
    	// 设置分页
        queryMap.put("from",0);
        queryMap.put("size",10);
        // testFieldValueFactor 就是上文定义的dsl模板名，queryMap 为查询条件，Item为实体类
    	//search_type=dfs_query_then_fetch 设置搜索类型 可参考相关资料的链接
        ESDatas<Item> esDatast = clientUtil.searchList("items/_search? search_type=dfs_query_then_fetch", "testFieldValueFactor", queryMap, Item.class);
        List<Item> esCrmOrderStudentList = esDatast.getDatas();
        logger.debug(esCrmOrderStudentList.toString());
        System.out.println(esCrmOrderStudentList.toString());
    }
```

实际执行返回的结果顺序如下：

```java
Items：
{title='雨伞', name='宜家', sales=1000}, //销量高的排行更高，评分为2.9443285
{title='雨伞', name='天堂伞', sales=500}
```

**注**

1. **`field_value_score`函数产生的分数必须是非负数**
2. **避免将log（n）设为0或负数等数学错误，会抛出异常**

## 2.4 random_score

这个函数的使用相当简单，只需要调用一下就可以返回一个 0 到 1 的分数（但不包括1）。为每个用户都使用一个不同的随机评分对结果排序，但对某一具体用户来说，排序规则保持一致。
它有一个非常有用的特性是可以通过seed属性设置一个随机种子，该函数保证在随机种子相同时返回值也相同，这点使得它可以轻松地实现对于用户的个性化推荐。

**注：**

1.  **可以在不设置字段的情况下设置种子，但这已被弃用，因为需要`_id`字段配置fielddata，这会消耗大量内存**

 	2.  **设置字段随机推荐时，该字段推荐是数值类型，设置其他类型字段也需要设置fielddata为true**

在esmapper/functionscore.xml定义一个名称为testRandomScore随机推荐id的dsl

```xml
<!--测试random_score-->
<property name="testRandomScore">
        <![CDATA[
		{
            "query": {
                "function_score": {
                    "random_score": {                       
                        "field": #[fieldName],
   						"seed": 10
                    }
                }
            },
            "from": #[from],
            "size": #[size]
        }
		]]>
</property>
```

执行上面的dsl：

```java
/**
* 测试RandomScore
*/
@Test
public void testRandomScoreDSL() {
        ClientInterface clientUtil = bbossESStarter.getConfigRestClient("esmapper/functionscore.xml");
        Map<String,Object> queryMap = new HashMap<>();
        // 指定进行random_score运算的字段,这里以id为随机给文档评分
        // 如果指定seed种子.seed相等返回值顺序相同，默认为null
        queryMap.put("fieldName","docId");
		// 设置分页
        queryMap.put("from",0);
        queryMap.put("size",10);
        // testRanodmScore就是上文定义的dsl模板名，queryMap 为查询条件，Student为实体类
        ESDatas<Student> esDatast =
                clientUtil.searchList("student/_search?search_type=dfs_query_then_fetch", "testRanodmScore", queryMap, 				Student.class);
        List<Student> esCrmOrderStudentList = esDatast.getDatas();
        logger.debug(esCrmOrderStudentList.toString());
        System.out.println(esCrmOrderStudentList.toString());
}
```

目前个人理解是给每一个文档一个随机的小于1的浮点数，并自动排序（排序规则也是随机的）。而seed相等的用户排序规则一致。

## 2.5 衰减函数decay functions

衰减函数（Decay Function）提供了一个更为复杂的公式，它描述了这样一种情况：对于一个字段，它有一个理想的值，而字段实际的值越偏离这个理想值（无论是增大还是减小），就越不符合期望。这个函数可以很好的应用于数值、日期和地理位置类型，由以下属性组成：

- 原点（origin）：该字段最理想的值，这个值可以得到满分（1.0）

- 偏移量（offset）：与原点相差在偏移量之内的值也可以得到满分

- 衰减规模（scale）：当值超出了原点到偏移量这段范围，它所得的分数就开始进行衰减了，衰减规模决定了这个分数衰减速度的快慢

- 衰减值（decay）：该字段可以被接受的值（默认为 0.5，范围是[0,1]），相当于一个分界点，具体的效果与衰减的模式有关

- 衰减函数只能用于数值，地理位置和日期场景。

  对于地理字段：可以定义为数字+单位（1km，12m，...）,默认单位是米，按照直径距离衰减
  对于日期字段：可以定义为数字+单位（“1h”，“10d”，...）,默认单位是毫秒
  对于数字字段：任何数值。

例如我们想要买一样东西：

- 它的理想价格是 50 元，这个值为原点

- 但是我们不可能非 50 元就不买，而是会划定一个可接受的价格范围，例如 45-55 元，±5 就为偏移量
- 当价格超出了可接受的范围，就会让人觉得越来越不值。如果价格是 70 元，评价可能是不太想买，而如果价格是 200 元，评价则会是不可能会买，这就是由衰减规模和衰减值所组成的一条衰减曲线

或者如果我们想租一套房：

- 它的理想位置是公司附近

- 如果离公司在 5km 以内，是我们可以接受的范围，在这个范围内我们不去考虑距离，而是更偏向于其他信息
- 当距离超过 5km 时，我们对这套房的评价就越来越低了，直到超出了某个范围就再也不会考虑了



衰减函数还可以指定三种不同的模式：线性函数（linear）、以 e 为底的指数函数（Exp）和高斯函数（gauss），它们拥有不同的衰减曲线

高斯函数（gauss）：

![](images\decay-function.png)

将上面提到的租房用 DSL 表示就是：

```json
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "公寓"
        }
      },
      "gauss": {
        "location": {
          "origin": { "lat": 40, "lon": 116 },
          "offset": "5km",
          "scale": "10km"
           }
         },
         "boost_mode": "sum"
    }
  }
}
```

我们希望租房的位置在`40, 116`坐标附近，`5km`以内是满意的距离，`15km`以内是可以接受的距离。

### 2.5.1 衰减函数Java实现

在bboss配置文件中esmapper/functionscore.xml定义一个名称为testRandomScore的根据距离推荐的租房用 DSL

```xml
<!--测试decayfunctions 地理类型-->
<property name="testDecayFunctionsByGeoPonit">
        <![CDATA[
            {
                "explain": false,
                "query": {
                "function_score": {
                  "query": {
                    "match": {
                      "title": #[titleName]
                    }
                  },
                  "gauss": {
                    "location": {
                        "origin": #[originLocation],
                        "offset": #[offset],
                        "scale": #[scale],
                        "decay": #[decay]
                       }
                     },
                     "boost_mode": "sum"
                  }
                },
                "from": #[from],
                "size": #[size]
            }
        ]]>
</property>
```

执行上面的dsl：

```java
/**
* 测试decayfunctions 地理类型
*/
@Test
public void testDecayFunctionsByGeoPonit() {
        ClientInterface clientUtil = bbossESStarter.getConfigRestClient("esmapper/functionscore.xml");
        Map<String,Object> queryMap = new HashMap<>();
        // 设置理想的商品名字
        queryMap.put("titleName","公寓");
        // origin 原点 设置北京的坐标
        queryMap.put("originLocation","40,116");
        // offset 理想范围
        queryMap.put("offset","3km");
        // scale 衰减临界点 注意该临界点为 origin±(offsetr+scale)
        queryMap.put("scale","10km");
        // 衰减系数 decay 乘以临界处文档的分数
        queryMap.put("decay",0.33);
     	// 设置分页
        queryMap.put("from",0);
        queryMap.put("size",10);
    
        ESDatas<Map> esDatast = clientUtil.searchList("hoses/_search?search_type=dfs_query_then_fetch","testDecayFunctionsByGeoPonit", queryMap, Map.class);
        List<Map> datas = esDatast.getDatas();
        logger.debug(datas.toString());
        System.out.println(datas.toString());
}
```

## 2.6 script_score

虽然强大的field_value_factor和衰减函数已经可以解决大部分问题了，但是也可以看出它们还有一定的局限性：

1. 这两种方式都只能针对一个字段计算分值
2. 这两种方式应用的字段类型有限，field_value_factor只用于数字类型，而衰减函数只用于数字、位置和时间类型

这时候就需要 script_score了，它支持我们自己编写一个脚本运行，在该脚本中我们可以拿到当前文档的字段信息，并返回一个指定分数从而实现根据字段信息评分。

举一个之前做不到的例子：

​	招生最想要招北京地区人大附中的学生，其他学校也可以，但是评分不高

之前的两种方式都无法给字符串打分，但是如果我们自己写脚本的话却很简单，使用 Groovy（Elasticsearch 的默认脚本语言）也就是一行的事：

```xml
return doc ['school.keyword'].value =='人大附中' ? 10 : 1.0
```

接下来只要将这个脚本配置到查询语句中就可以了：

```xml
<!--测试script_score-->
<property name="testScriptScore">
        <![CDATA[
		{
			"explain": false,
            "query": {
              "function_score": {
                "query": {
                  "match": {
                    "city": #[cityName]
                  }
                },
                "script_score": {
                  "script": "return doc ['school.keyword'].value == '#[schoolName,quoted=false]' ? 10 : 1.0"
                }
              }
            },
            "from": #[from],
            "size": #[size]
        }
		]]>
</property>
```

执行上面的dsl：

```java
/**
* 测试script_score
*/
@Test
public void testScriptScore() {
        ClientInterface clientUtil = bbossESStarter.getConfigRestClient("esmapper/functionscore.xml");
        Map<String,Object> queryMap = new HashMap<>();
        queryMap.put("cityName","北京");
        queryMap.put("schoolName","人大附中");
        // 设置分页
        queryMap.put("from",0);
        queryMap.put("size",10);
    
        ESDatas<Student> esDatast = clientUtil.searchList("student/_search?search_type=dfs_query_then_fetch", "testScriptScore",queryMap,Student.class);
        List<Student> esCrmOrderStudentList = esDatast.getDatas();
        logger.debug(esCrmOrderStudentList.toString());
        System.out.println(esCrmOrderStudentList.toString());
}
```

当然我们可以创建一个脚本函数，然后在查询语句中引用它

在配置文件中添加创建脚本函数的dsl定义-schoolScoreScript

```xml
<!--script 脚本引用-->
<property name="schoolScoreScript">
        <![CDATA[{
            "script": {
              "lang": "painless",
              "source": @"""
                return doc ['school.keyword'].value == params.schoolName ? 10.0 : 1.0;
              """
            }
        }]]>
</property>
```

通过bboss提供的api在elasticsearch中建立名称为schoolScoreScript的脚本函数，通过id调用该脚本。

```java
@Test
public void testCreateSchoolScoreScript() {
        ClientInterface clientUtil = bbossESStarter.getConfigRestClient("esmapper/functionscore.xml");
        //创建评分脚本函数testScriptScore
        clientUtil.executeHttp("_scripts/schoolScoreScript", "schoolScoreScript",
                ClientInterface.HTTP_POST);
        //获取刚才创建评分脚本函数testScriptScore
        String schoolScoreScript = clientUtil.executeHttp("_scripts/schoolScoreScript",
                ClientInterface.HTTP_GET);
        System.out.println(schoolScoreScript);
}
```

接下来在dsl配置文件中定义一条采用id为schoolScoreScript的脚本函数来实现根据学校挑选学生dsl语句：

```xml
<!--测试script_score 脚本-->
<property name="testScriptScoreByIncloudScript">
        <![CDATA[
		{
            "query": {
			  "explain": false,
              "function_score": {
                "query": {
                  "match": {
                    "city": #[cityName]
                  }
                },
                "script_score": {
                  "script": {
                    "id": "schoolScoreScript",
                     "params": {
                         "schoolName":#[schoolName]  ## 传入评分脚本函数需要的学校名称
                     }
                  }
                }
              }
            },
            "from": #[from],
            "size": #[size]
        }
		]]>
</property>
```

执行检索操作：

```java
@Test
public void testScriptScoreByIncloudScript() {
        ClientInterface clientUtil = bbossESStarter.getConfigRestClient("esmapper/functionscore.xml");
        Map<String,Object> queryMap = new HashMap<>();
        queryMap.put("cityName","北京");
        queryMap.put("schoolName","人大附中");
        // 设置分页
        queryMap.put("from",0);
        queryMap.put("size",10);
    
        ESDatas<Student> esDatas = clientUtil.searchList("student/_search?search_type=dfs_query_then_fetch","testScriptScoreByIncloudScript",queryMap,Student.class);
        List<Student> students = esDatas.getDatas();
        System.out.println(students);
}
```

## 2.7 同时使用多个函数

​	上面的例子都只是调用某一个函数并与查询得到的_score进行合并处理，而在实际应用中肯定会出现在多个点上计算分值并合并，虽然脚本也许可以解决这个问题，但是应该没人愿意维护一个复杂的脚本吧。这时候通过多个函数将每个分值都计算出在合并才是更好的选择。

​	在 function_score中可以使用functions属性指定多个函数。它是一个数组，所以原有函数不需要发生改动。同时还可以通过score_mode指定各个函数分值之间的合并处理，值跟最开始提到的boost_mode相同。下面举两个例子介绍一些多个函数混用的场景。

### 2.7.1 大众点评的餐厅应用

​	第一个例子是类似于大众点评的餐厅应用。该应用希望向用户推荐一些不错的餐馆，特征是：范围要在当前位置的 5km 以内，有停车位是最重要的，餐厅的评分（1 分到 5 分）越高越好，并且对不同用户最好展示不同的结果以增加随机性。

那么它的查询语句应该是这样的：

```xml
<!--餐厅评分 综合函数-->
<property name="testHellFunctionScore">
        <![CDATA[
        {
            "explain": false,
            "query": {
                "function_score": {
                    "query": {
                        "bool": {
                          "filter": {
                            "geo_distance": {
                              "distance": "10km",
                              "location": {
                                "lat": 40,
                                "lon": 116
                              }
                            }
                          }
                        }
                    },
                    "functions": [
                    {
                      "filter": {
                        "match": {
                          "features": #[features]
                        }
                      },
                      "weight": 10
                    },
                    {
                        "field_value_factor": {
                           "field": #[valueFactorFieldName],
                           "factor": 0.1
                         }
                    },
                    {
                      "random_score": {
                        "field":"docId",
                        "seed": #[docId]
                      }
                    }
                    ],
                ## 各个函数分数的组合方式
                "score_mode": "sum",
                ## 评分得分和文档分数的组合方式
                "boost_mode": "sum"
                }
            },
            "from": #[from],
            "size": #[size]
        }
        ]]>
</property>
```

餐厅检索的java实现：

```java
/**
* 测试 餐厅评分 FunctionScore
*/
@Test
public void testFunctionScore() {
        ClientInterface clientUtil = bbossESStarter.getConfigRestClient("esmapper/functionscore.xml");
        Map<String,Object> queryMap = new HashMap<>();
        queryMap.put("features","停车位");
        queryMap.put("valueFactorFieldName","score");
        queryMap.put("originLocation","40,116");
        queryMap.put("scale","10km");
        queryMap.put("docId",15);
        // 设置分页
        queryMap.put("from",0);
        queryMap.put("size",10);

        ESDatas<Map> esDatast = clientUtil.searchList("hell/_search", "testFunctionScore",queryMap,Map.class);
        List<Map> datas = esDatast.getDatas();
        logger.debug(datas.toString());
        System.out.println(datas.toString());
    }

```

这样一个餐厅的最高得分应该是 2 分（有停车位）+ 1 分（有 wifi）+ 6 分（评分 5 分 \* 1.2）+ 1 分（随机评分）。

### 2.7.2 新浪微博的社交网站

另一个例子是类似于新浪微博的社交网站。现在要优化搜索功能，使其以文本相关度排序为主，但是越新的微博会排在相对靠前的位置，点赞（忽略相同计算方式的转发和评论）数较高的微博也会排在较前面。如果这篇微博购买了推广并且是创建不到 24 小时（同时满足），它的位置会非常靠前。

```json
<!--新浪微博评分 综合函数-->
<property name="testSinaFunctionScore">
        <![CDATA[
        {
            "query": {
                "function_score": {
                    "query": {
                        "match": {
                            "content": #[content]
                        }
                    },
                    "functions": [
                        {
                            "gauss": {
                                  "createDate": {
                                      "origin": #[createDate],
                                      "scale": "6d",
                                      "offset": "1d"
                                }
                            }
                        },
                        {
                            "field_value_factor": {
                                "field": #[valueFactorFieldName],
                                "modifier": "log1p",
                                "factor": 0.1
                            }
                        },
                        {
                            "script_score": {
                              "script":
                                   "return doc['is_recommend'].value && doc['is_recommend'].value && doc['createDate'].value.getMillis() > new Date().getTime()  ? 1.5 : 1.0"
                            }
                        }
                    ],
                    "boost_mode": "multiply"
                    }
                }
            },
            "from": #[from],
            "size": #[size]
        }
        ]]>
</property>
```

它的公式为：

```java
_score * gauss (create_date, $now, "1d", "6d") * log (1 + 0.1 * like_count) * is_recommend ? 1.5 : 1.0
```

执行上面的dsl：

```
/**
* 测试 新浪微博评分 FunctionScore
*/
@Test
public void testSinaFunctionScore() {
        ClientInterface clientUtil = bbossESStarter.getConfigRestClient("esmapper/functionscore.xml");
        Map<String,Object> queryMap = new HashMap<>();
        queryMap.put("content","刘亦菲");

        Date date = new Date(); //获取当前的系统时间。
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd") ; //使用了默认的格式创建了一个日期格式化对象。
        String time = dateFormat.format(date); //可以把日期转换转指定格式的字符串

        queryMap.put("createDate",time);
        queryMap.put("valueFactorFieldName","like_count");
        queryMap.put("time",new Date().getTime());
        // 设置分页
        queryMap.put("from",0);
        queryMap.put("size",10);

        ESDatas<Map> esDatast = clientUtil.searchList("xinlang/_search?search_type=dfs_query_then_fetch", "testSinaFunctionScore",queryMap,Map.class);
        List<Map> datas = esDatast.getDatas();
        logger.debug(datas.toString());
        System.out.println(datas.toString());
}
```



# 3.相关资料

https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html

https://esdoc.bbossgroups.com/#/development?id=_53-dsl%E9%85%8D%E7%BD%AE%E8%A7%84%E8%8C%83

评分相似文档

https://blog.csdn.net/wwd0501/article/details/78652850?tdsourcetag=s_pcqq_aiomsg

设置search_type

https://esdoc.bbossgroups.com/#/development?id=_49-%E6%8C%87%E5%AE%9A%E6%A3%80%E7%B4%A2search_type%E5%8F%82%E6%95%B0

# 4.开发交流



bboss elasticsearch交流：166471282

**bboss elasticsearch微信公众号：**

<img src="https://static.oschina.net/uploads/space/2017/0617/094201_QhWs_94045.jpg"  height="200" width="200">



