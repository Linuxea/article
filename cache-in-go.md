## Use Cache With Go



缓存的使用在后端系统中非常普遍，可以通过缓存提高响应速度，减少复杂计算所消耗的资源，缓解数据库压力等等。

同时缓存的引入也会带来一些新的问题，缓存的不合理使用，缓存雪崩与击穿，数据倾斜，资源占用高命中率低，模板代码与业务代码混杂等等。

本文主要想讲讲代码中如何以友好的方式与 cache 并存。




## Question In Go

使用 `java` 或者其他一些语言，能够通过类注解的方式以动态代理的实现轻松对原有逻辑进行增强。

但是 Go 中没有类似注解的手段，这让不少人可能在一开始有点无从下手。

代码中充斥着很多模板化的代码：判断缓存有没有具体的某些数据，有则直接返回，没有则从数据库获取，手动保存到缓存，最终进行返回。

为了消除这些重复的代码，我们需要使用 Go 来提供一个统一的入口。

这跟上篇文章中讲到的 web 中间件处理入口是一致的理念。

同时在 `GORM` 中框架为我们提供的 `callback` 函数注册回调也是一样的。

我们需要一个入口来帮助我们实现统一的处理。



## Code

我们可以通过反射来实现应付多种数据结构的缓存

以下是具体的实现代码

```golang
package cache

import (
	"encoding/json"
	"reflect"
	"time"

	"github.com/go-redis/redis"
)

type Cache struct {
	key      string                      // redis key
	redis    *redis.Client               // redis cli
	model    interface{}                 // the data model instance, require reflect.Type from it
	FetchFun func() (interface{}, error) // execute in case of cache does not exists
	ttl      time.Duration               // cache ttl
}

// List list cache
func (c *Cache) List() (interface{}, error) {
	return c.cache(true)
}

// Get get cache
func (c *Cache) Get() (interface{}, error) {
	return c.cache(false)
}

func (c *Cache) cache(sliceOrArray bool) (interface{}, error) {

    // has cache already ?
	s, err := c.redis.Get(c.key).Result()
	if err != nil && err != redis.Nil {
		return nil, err
	}

	// from cache
	if err == nil {
		var result interface{}
		if sliceOrArray {
			result = reflect.New(reflect.SliceOf(reflect.TypeOf(c.model))).Interface()
		} else {
			result = reflect.New(reflect.TypeOf(c.model)).Interface()
		}

		if err = json.Unmarshal([]byte(s), result); err != nil {
			return nil, err
		}
		return reflect.ValueOf(result).Elem().Interface(), nil
	}

    // from realtime execute

	fetchData, err := c.FetchFun()
	if err != nil {
		return nil, err
	}
	// put into cache
	b, err := json.Marshal(fetchData)
	if err != nil {
		return nil, err
	}
	_, err = c.redis.Set(c.key, b, c.ttl).Result()
	if err != nil {
		return nil, err
	}

	return fetchData, nil
}
```


在代码中定义了结构体 `Cache`， 里面包含了我们所需要的一切信息。
- 缓存的唯一的 `key`
- `redis` 客户端连接
- `model` 为对应的数据结构，通过反射来获取 `reflect.Type` 来转换返回的数据类型
- `ttl` 缓存的自定义过期时间
- 以及最重要的 `FetchFun`，通过注入获取数据函数，来跟真实的数据库或者其他数据层相关联操作进行解耦



主要的缓存逻辑存在于 `cache` 内部方法中。
- 判断是否存在对应缓存
- 如果存在缓存，根据 `model` 通过反射来创建结构体或者切片来保存反序列化后的数据
- 如果不存在缓存则执行 `FetchFun` 函数获取数据，保存进缓存，并返回。



## Test


接下来我们创建几个不同类型的数据类型来完善我们的单元测试

```golang
package cache

import (
	"fmt"
	"log"
	"os"
	"testing"
	"time"

	"github.com/go-redis/redis"
	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

var db *gorm.DB
var redisCli *redis.Client

func init() {
	var err error
	db, err = gorm.Open(sqlite.Open("test.db"), &gorm.Config{
		Logger: logger.New(log.New(os.Stdout, "\r\n", log.LstdFlags), logger.Config{
			LogLevel: logger.Info,
		}),
	})
	if err != nil {
		panic("failed to connect database")
	}
}

func init() {
	redisCli = redis.NewClient(&redis.Options{
		PoolSize: 10,
		Addr:     "172.17.0.1:6379",
		Password: "",
		DB:       0,
	})
}

func TestList(t *testing.T) {

	type Product struct {
		gorm.Model
		Code  string
		Price uint
	}

	db.AutoMigrate(&Product{})

	// list
	cache := &Cache{key: "productAll",
		redis: redisCli,
		model: &Product{},
		FetchFun: func() (interface{}, error) {
			list := make([]*Product, 0)
			return list, db.Find(&list).Error
		},
		ttl: time.Second * 10,
	}
	li, err := cache.List()
	if err != nil {
		panic(err)
	}

	if p, ok := li.([]*Product); ok {
		for _, temp := range p {
			fmt.Printf("%d 价格对应为 %d\n", temp.ID, temp.Price)
		}

	} else {
		fmt.Println("transfrom error")
	}

}

func TestGetString(t *testing.T) {
	// get string
	getStringCache := &Cache{key: "randomString",
		redis: redisCli,
		model: "",
		FetchFun: func() (interface{}, error) {
			return time.Now().Format("2006-01-02 15:04:05"), nil
		},
		ttl: time.Second * 10,
	}

	getRes, err := getStringCache.Get()
	if err != nil {
		panic(err)
	}

	fmt.Println(getRes)
}

func TestGetInt(t *testing.T) {
	// get int
	getIntCache := &Cache{key: "randomInt",
		redis: redisCli,
		model: 0,
		FetchFun: func() (interface{}, error) {
			return time.Now().Second(), nil
		},
		ttl: time.Second * 10,
	}

	getIntRes, err := getIntCache.Get()
	if err != nil {
		panic(err)
	}

	fmt.Println(getIntRes.(int))
}

func TestGetStruct(t *testing.T) {

	type Product struct {
		gorm.Model
		Code  string
		Price uint
	}

	// get struct
	getOneProductCache := &Cache{key: "oneProduct",
		redis: redisCli,
		model: &Product{},
		FetchFun: func() (interface{}, error) {
			return &Product{Code: "abcdefg", Price: 1000}, nil
		},
		ttl: time.Second * 10,
	}

	i, err := getOneProductCache.Get()
	if err != nil {
		panic(err)
	}

	p, ok := i.(*Product)
	if ok {
		fmt.Println(p.Code, p.Price)
	}
}

func TestGetComplicateStruct(t *testing.T) {

	type Product struct {
		gorm.Model
		Code  string
		Price uint
	}

	type Person struct {
		Products []*Product
		Name     string
		Age      int
	}

	// get Person
	getPersonCache := &Cache{key: "getPerson",
		redis: redisCli,
		model: &Person{},
		FetchFun: func() (interface{}, error) {
			return &Person{
				Products: []*Product{{Code: "abcdefg", Price: 1000}},
				Name:     "linuxea",
				Age:      12,
			}, nil
		},
		ttl: time.Second * 30,
	}

	i2, err := getPersonCache.Get()
	if err != nil {
		panic(err)
	}
	p2 := i2.(*Person)
	fmt.Println(p2)
}

```

```bash
➜  go test -v go.tour/linuxea/cache
=== RUN   TestList

2022/01/11 23:23:43 /home/linuxea/go/pkg/mod/gorm.io/driver/sqlite@v1.2.6/migrator.go:32
[0.035ms] [rows:-] SELECT count(*) FROM sqlite_master WHERE type='table' AND name="products"

2022/01/11 23:23:43 /home/linuxea/Desktop/code/go-tour/cache/cache_test.go:48
[0.028ms] [rows:-] SELECT * FROM `products` LIMIT 1

2022/01/11 23:23:43 /home/linuxea/go/pkg/mod/gorm.io/driver/sqlite@v1.2.6/migrator.go:257
[0.037ms] [rows:-] SELECT count(*) FROM sqlite_master WHERE type = "index" AND tbl_name = "products" AND name = "idx_products_deleted_at"

2022/01/11 23:23:43 /home/linuxea/Desktop/code/go-tour/cache/cache_test.go:56
[0.129ms] [rows:0] SELECT * FROM `products` WHERE `products`.`deleted_at` IS NULL
--- PASS: TestList (0.00s)
=== RUN   TestGetString
2022-01-11 23:23:43
--- PASS: TestGetString (0.00s)
=== RUN   TestGetInt
43
--- PASS: TestGetInt (0.00s)
=== RUN   TestGetStruct
abcdefg 1000
--- PASS: TestGetStruct (0.00s)
=== RUN   TestGetComplicateStruct
&{[0xc0001f7180] linuxea 12}
--- PASS: TestGetComplicateStruct (0.00s)
PASS
ok      go.tour/linuxea/cache   0.008s
```



上述测试文件中，我们覆盖了测试场景：
- 从数据库获取数据
- 直接内存数据列表
- 返回简单结构体
- 返回字符串
- 返回数据
- 返回复杂结构体




## Conclusion

通过这种方式可以节省很多模板代码，当你发现项目中存在许多类似的模板化代码，也许是个改变的契机。


## 参考

- [1] [standard lib reflect](https://pkg.go.dev/reflect)
- [2] [Using Dependency Inversion in Go](https://medium.com/itnext/using-dependency-inversion-in-go-31d8bf9b3760)
- [3] [The fantastic ORM library for Golang, aims to be developer friendly](https://github.com/go-gorm/gorm)