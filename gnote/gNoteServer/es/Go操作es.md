下载olivere/elastic
```Go
go get github.com/olivere/elastic/v7
```

# 连接es
```yaml
es:
    addr: http://localhost:9200
    user: elastic
    password:
```
在config映射配置中定义es的配置：
```go
// config_es.go

type ES struct {
	Addr     string `yaml:"addr"`
	User     string `yaml:"user"`
	Password string `yaml:"password"`
}
```

```go
// init_es.go
func InitES() (client *elastic.Client) {
	es := global.Config.ES
	addr := es.Addr
	client, err := elastic.NewClient(
		elastic.SetURL(addr),
		elastic.SetSniff(false),
		elastic.SetBasicAuth(es.User, es.Password),
	)
	if err != nil {
		logrus.Fatalf(fmt.Sprintf("[%s] es连接失败, err:%s", addr, err.Error()))
		panic(err)
	}
	return client
```
- SetURL：指定es地址
- SetSniff：这里false关闭节点嗅探功能，因为这是一个单节点部署
- SetBasicAuth：设置用户名密码

# 全文搜索模型
这里的全文搜索是在命令行创建的
```go
// flags_es_index.go

func ESIndex() {
  indexs.CreateIndex(models.FullTextModel{})
}
```

```go
// full_text_model.go

type ESIndexInterface interface {
  Index() string
  Mapping() string
}

// 全文搜索模型，FullTextModel实现了这个功能
type FullTextModel struct {
	DocID uint   `json:"docID"`
	ID    uint   `json:"id"` // es ID（连带查出来的，赋值不需要）
	Title string `json:"title"`
	Body  string `json:"body"`
	Slug  string `json:"slug"` // 跳转地址，由docID和title组成
}

func (FullTextModel) Index() string {
	return "gn_server_full_text_index"
}

func (FullTextModel) Mapping() string {
	return `
{
  "mappings": {
    "properties": {
      "body": { 
        "type": "text" 
      },
      "title": {
        "type": "text",
	    "fields": {
          "keyword": {
              "type": "keyword",
			  "ignore_above": 256
          }
        }
	  },
	  "slug": { 
        "type": "keyword" 
      },
      "docID": {
        "type": "integer"
      }
    }
  }
}
`
}
```

# 操作
## 索引操作（service/es_service）
### CreateIndex 创建索引
```go
func CreateIndex(esIndexInterface models.ESIndexInterface) {
	if global.ESClient == nil {
		logrus.Fatalf("请配置es连接")
	}
	index := esIndexInterface.Index()
	if ExistsIndex(index) {
		DeleteIndex(index)
	}

	createIndex, err := global.ESClient.
		CreateIndex(index).
		BodyString(esIndexInterface.Mapping()).Do(context.Background())
	if err != nil {
		logrus.Fatalf("%s err: %s", index, err)
	}

	logrus.Infof("索引 %s 创建成功", createIndex.Index)
}
```
### ExistsIndex 判断索引是否存在
```go
func ExistsIndex(index string) bool {
	if global.ESClient == nil {
		logrus.Fatalf("请配置es连接")
	}

	exists, _ := global.ESClient.IndexExists(index).Do(context.Background())
	return exists
}
```

### DeleteIndex 删除索引
```go
func DeleteIndex(index string) {
	if global.ESClient == nil {
		logrus.Fatalf("请配置es连接")
	}

	_, err := global.ESClient.
		DeleteIndex(index).Do(context.Background())
	if err != nil {
		logrus.Fatalf("%s err: %s", index, err)
	}
	fmt.Println(index, "索引删除成功")
	logrus.Infof("索引 %s 删除成功", index)
}
```
## 文档操作（service/full_search_service)
### FullSearchCreate文档添加
```go
// full_search_create.go
func FullSearchCreate(doc models.DocModel) {
	if global.ESClient == nil {
		return
	}

	searchDataList := MarkdownParse(doc.ID, doc.Title, doc.Content)
	// 创建批量操作对象，Refresh("true")表示写入索引后立即刷新索引
	bulk := global.ESClient.Bulk().Index(models.FullTextModel{}.Index()).Refresh("true")
	for _, model := range searchDataList {
		req := elastic.NewBulkCreateRequest().Doc(models.FullTextModel{
			DocID: doc.ID,
			Title: model.Title,
			Body:  model.Body,
			Slug:  model.Slug,
		})
		bulk.Add(req)
	}
	res, err := bulk.Do(context.Background())
	if err != nil {
		fmt.Println(err)
		logrus.Errorf("%#v 数据添加失败 err:%s", doc, err.Error())
		return
	}
	logrus.Infof("添加全文搜集记录 %d 条", len(res.Succeeded()))
}
```

### FullSearchDelete文档删除（根据id删）
```go
// full_search_delete.go

func FullSearchDelete(docID uint) {
	if global.ESClient == nil {
		return
	}
	// 构造查询条件，term，表示查找所有docID等于传入参数的文档
	var query = elastic.NewTermQuery("docID", docID)
	// 条件批量删除
	res, err := global.ESClient.DeleteByQuery(models.FullTextModel{}.Index()).
		Query(query).Refresh("true").Do(context.Background())
	if err != nil {
		logrus.Errorf("%d 数据删除失败 err:%s", docID, err.Error())
		return
	}
	logrus.Infof("删除全文搜索记录 %d 条", res.Deleted)
}

```

### FullSearchUpate文档创建（覆盖）
```go
// full_search_update.go
// FullSearchUpdate 更新，先删除再添加
func FullSearchUpdate(doc models.DocModel) {
	if global.ESClient == nil {
		return
	}
	// 添加之前先删除之前的
	FullSearchDelete(doc.ID)
	FullSearchCreate(doc)
}
```
### Markdown转换
文档转换问题：
1. 一开始就是正文
```Markdown
正文1
# 标题1
正文2
# 标题2
```
2. 没有标题
```Markdown
正文
```
3. 标题下没有正文
```Markdown
# 标题1
## 标题2
正文
```
4. 代码块注释问题（如Python）
```Markdown
# 标题
正文
```Python
# 注释
```

==因为文档一定会有文档标题==  
所以第一、第二问题不存在  
```go
// markdown_parse.go
package full_search_service

import (
	"fmt"
	"strings"
)

type SearchData struct {
	Title string `json:"title"`
	Body  string `json:"body"`
	Slug  string `json:"slug"`
}

// MarkdownParse 解析md正文，传递文档标题，正文
func MarkdownParse(id uint, title, content string) (searchDataList []SearchData) {
	var body string
	var headList []string
	var bodyList []string
	var isCode bool
	list := strings.Split(content, "\n")
	// 标题
	headList = append(headList, title)
	for _, s := range list {
		if strings.Contains(s, "```") {
			isCode = !isCode
		}
		if strings.HasPrefix(s, "#") && !isCode {
			headList = append(headList, getHead(s))
			bodyList = append(bodyList, body)
			body = ""
			continue
		}
		body += s
	}
	bodyList = append(bodyList, body)
	ln := len(headList)
	for i := 0; i < ln; i++ {
		searchDataList = append(searchDataList, SearchData{
			Title: headList[i],
			Body:  bodyList[i],
			Slug:  getSlug(id, headList[i]),
		})
		fmt.Printf("%#v\n", searchDataList)
	}
	return
}

func getHead(head string) string {
	head = strings.ReplaceAll(head, "#", "")
	head = strings.ReplaceAll(head, " ", "")
	return head
}

func getSlug(id uint, title string) string {
	// #是前端锚点
	return fmt.Sprintf("%d#%s", id, title)
}
```

# 全文搜索
```go
// doc_api/doc_search.go
func (DocApi) DocSearchView(c *gin.Context) {
	var cr models.Pagination
	_ = c.ShouldBindQuery(&cr)

	token := c.Request.Header.Get("token")
	fmt.Println("token:", token)
	claims, err := jwts.ParseToken(token)

	var roleID uint = 2 // 默认访客
	if err == nil {
		roleID = claims.RoleID
	}

	fmt.Println("roleID:", roleID)
	var docIDList []uint
	global.DB.Model(models.RoleDocModel{}).Where("role_id =?", roleID).Select("doc_id").Scan(&docIDList)

	if global.ESClient == nil {
		res.FailWithMsg("请配置es", c)
		return
	}

	if cr.Limit == 0 || cr.Limit > 100 {
		cr.Limit = 10
	}
	if cr.Page == 0 {
		cr.Page = 1
	}
	from := (cr.Page - 1) * cr.Limit

	// 构造组合查询
	var query = elastic.NewBoolQuery()

	if cr.Key != "" {
		query.Must(elastic.NewMultiMatchQuery(cr.Key, "title", "body"))
	}

	var ids []interface{}
	for _, docID := range docIDList {
		ids = append(ids, docID)
	}
	// 用户只能查询自己有权限的文档
	query.Must(elastic.NewTermsQuery("docID", ids...))

	// 1. 查询 2. 高亮
	result, err := global.ESClient.Search(models.FullTextModel{}.Index()).
		Query(query).
		Highlight(elastic.NewHighlight().Field("body").Field("title")).
		From(from).Size(cr.Limit).Do(context.Background())
	if err != nil {
		logrus.Error(err)
		res.FailWithMsg("查询失败", c)
		return
	}
	count := result.Hits.TotalHits.Value
	var list = make([]models.FullTextModel, 0)
	for _, hit := range result.Hits.Hits {
		var model models.FullTextModel
		err = json.Unmarshal(hit.Source, &model)

		// 高亮
		bodyList, ok := hit.Highlight["body"]
		if ok {
			model.Body = bodyList[0]
		}
		titleList, ok := hit.Highlight["title"]
		if ok {
			model.Title = titleList[0]
		}

		list = append(list, model)
	}
	res.OKWithList(list, int(count), c)
}

```