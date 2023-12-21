## Redis 基本数据结构详解

### String

1. 底层数据实现是**SDS**($Simple Dynamic String$),优点有：相比于 C 的原生字符串，Redis 的 SDS 不光可以保存文本数据还可以保存二进制数据，并且获取字符串长度复杂度为 O(1)，并且不会出现缓冲区溢出。

2. 常用的命令有：

   > | 命令                           | 介绍                             |
   > | ------------------------------ | -------------------------------- |
   > | SET key value                  | 设置指定 key 的值                |
   > | SETNX key value                | 只有在 key 不存在时设置 key 的值 |
   > | GET key                        | 获取指定 key 的值                |
   > | INCR key                       | 将 key 中储存的数字值增一        |
   > | DECR key                       | 将 key 中储存的数字值减一        |
   > | EXISTS key                     | 判断指定 key 是否存在            |
   > | DEL key（通用）                | 删除指定的 key                   |
   > | EXPIRE key seconds（通用）     | 给指定 key 设置过期时间          |
   > | MSET key1 value1 key2 value2 … | 设置一个或多个指定 key 的值      |
   > | MGET key1 key2 ...             | 获取一个或多个指定 key 的值      |
   > | STRLEN key                     | 返回 key 所储存的字符串值的长度  |

3. 基本应用场景

   **需要存储常规数据的场景**

   - 举例：缓存 session、token、图片地址、序列化后的对象(相比较于 Hash 存储更节省内存)。
   - 相关命令：`SET`、`GET`。

   **需要计数的场景**

   - 举例：用户单位时间的请求数（简单限流可以用到）、页面单位时间的访问数。
   - 相关命令：`SET`、`GET`、 `INCR`、`DECR` 。

   **分布式锁**

   利用 `SETNX key value` 命令可以实现一个最简易的分布式锁（存在一些缺陷，通常不建议这样实现分布式锁）。

4. 代码实例：

   > ```go
   > package main
   > 
   > import (
   > 	"net/http"
   > 	"time"
   > 
   > 	"github.com/gin-gonic/gin"
   > 	"github.com/go-redis/redis/v8"
   > )
   > 
   > var redisClient *redis.Client
   > 
   > type Data struct {
   > 	Key   string `from:"key",json:"key"`
   > 	Value string `from:"key",json:"value"`
   > }
   > 
   > func main() {
   > 	redisClient = redis.NewClient(&redis.Options{
   > 		Addr:     "localhost:6379",
   > 		Password: "",
   > 		DB:       0,
   > 	})
   > 	r := gin.Default()
   > 	r.POST("/set/", setStringHandler)
   > 	r.GET("/get", getStringHandler)
   > 	r.POST("/delete", deleteStringHandler)
   > 	r.POST("/incr", incrStringHandler)
   > 	r.POST("/decr", decrStringHandler)
   > 	r.POST("/exist", existStringHandler)
   > 	r.POST("/expire", expireStringHandler)
   > 	r.POST("/mset", mSetStringHandler)
   > 	r.POST("/mget", mGetStringHandler)
   > 	r.Run(":8080")
   > }
   > func mSetStringHandler(c *gin.Context) {
   > 	var kvs map[string]string
   > 	err := c.ShouldBindJSON(&kvs)
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "MSet请求数据绑定失败！！！！！",
   > 		})
   > 		return
   > 	}
   > 
   > 	var pairs []interface{}
   > 	for k, v := range kvs {
   > 		pairs = append(pairs, k, v)
   > 	}
   > 
   > 	err = redisClient.MSet(c, pairs...).Err()
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "MSet数据批处理绑定失败！！！！！",
   > 		})
   > 		return
   > 	}
   > 	c.JSON(http.StatusOK, gin.H{
   > 		"message": "MSet 批处理数据成功！！！！",
   > 	})
   > }
   > func mGetStringHandler(c *gin.Context) {
   > 	var keys []string
   > 	err := c.ShouldBindJSON(&keys)
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "MGet请求数据绑定失败！！！！！",
   > 		})
   > 		return
   > 	}
   > 
   > 	res, err := redisClient.MGet(c, keys...).Result()
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "MGet数据批处理绑定失败！！！！！",
   > 		})
   > 		return
   > 	}
   > 	keyValueMap := make(map[string]interface{})
   > 	for i, key := range keys {
   > 		keyValueMap[key] = res[i]
   > 	}
   > 	c.JSON(http.StatusOK, gin.H{
   > 		"message": "MGet 批处理数据成功！！！！",
   > 		"data":    keyValueMap,
   > 	})
   > }
   > 
   > func expireStringHandler(c *gin.Context) {
   > 	key := c.PostForm("key")
   > 	durationStr := c.PostForm("duration")
   > 	duration, err := time.ParseDuration(durationStr)
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "duration 解析失败！！！！",
   > 		})
   > 		return
   > 	}
   > 	_, err = redisClient.Expire(c, key, duration).Result()
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "设置过期时间失败！！！！",
   > 		})
   > 		return
   > 	}
   > 	c.JSON(http.StatusOK, gin.H{
   > 		"message": "String 数据设置过期时间成功！！！！",
   > 		//"expire":  fmt.Sprintf("%t", res),
   > 	})
   > }
   > func existStringHandler(c *gin.Context) {
   > 	key := c.PostForm("key")
   > 	nums, err := redisClient.Exists(c, key).Result()
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "所请求的查询失败！！！！",
   > 		})
   > 		return
   > 	}
   > 	c.JSON(http.StatusOK, gin.H{
   > 		"nums":    nums,
   > 		"message": "数据查询成功！！！！",
   > 	})
   > 
   > }
   > func incrStringHandler(c *gin.Context) {
   > 	key := c.PostForm("key")
   > 
   > 	err := redisClient.Incr(c, key).Err()
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "所请求的数据新增1失败！！！！",
   > 		})
   > 		return
   > 	}
   > 	c.JSON(http.StatusOK, gin.H{
   > 		"message": "所请求的数据新增1成功！！！！",
   > 	})
   > }
   > func decrStringHandler(c *gin.Context) {
   > 	key := c.PostForm("key")
   > 
   > 	err := redisClient.Decr(c, key).Err()
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "所请求的数据减少1失败！！！！",
   > 		})
   > 		return
   > 	}
   > 	c.JSON(http.StatusOK, gin.H{
   > 		"message": "所请求的数据减少1成功！！！！",
   > 	})
   > }
   > func deleteStringHandler(c *gin.Context) {
   > 	key := c.PostForm("key")
   > 
   > 	err := redisClient.Del(c, key).Err()
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "String数据删除失败！！！！",
   > 		})
   > 		return
   > 	}
   > 	c.JSON(http.StatusOK, gin.H{
   > 		"message": "String数据删除成功！！！",
   > 	})
   > }
   > func setStringHandler(c *gin.Context) {
   > 	var data = Data{}
   > 	if err := c.ShouldBindJSON(&data); err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "数据绑定失败！！！",
   > 		})
   > 		return
   > 	}
   > 	err := redisClient.Set(c, data.Key, data.Value, 0).Err()
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "String类型数据插入失败！！！",
   > 		})
   > 		return
   > 	}
   > 	c.JSON(http.StatusOK, gin.H{
   > 		"message": "String类型数据插入成功！！！！",
   > 	})
   > 
   > }
   > func getStringHandler(c *gin.Context) {
   > 	key := c.Query("key")
   > 
   > 	value, err := redisClient.Get(c, key).Result()
   > 
   > 	if err != nil {
   > 		c.JSON(http.StatusBadRequest, gin.H{
   > 			"error":   err.Error(),
   > 			"message": "String数据查询失败！！！",
   > 		})
   > 		return
   > 	}
   > 	c.JSON(http.StatusOK, gin.H{
   > 		"key":     key,
   > 		"value":   value,
   > 		"message": "String类型数据查询成功！！！",
   > 	})
   > }
   > 
   > ```
   >
   > 

### List

1. 底层实现为：LinkedList/ZipList/QuickList

2. 常用命令有：

   > | 命令                        | 介绍                                       |
   > | --------------------------- | ------------------------------------------ |
   > | RPUSH key value1 value2 ... | 在指定列表的尾部（右边）添加一个或多个元素 |
   > | LPUSH key value1 value2 ... | 在指定列表的头部（左边）添加一个或多个元素 |
   > | LSET key index value        | 将指定列表索引 index 位置的值设置为 value  |
   > | LPOP key                    | 移除并获取指定列表的第一个元素(最左边)     |
   > | RPOP key                    | 移除并获取指定列表的最后一个元素(最右边)   |
   > | LLEN key                    | 获取列表元素数量                           |
   > | LRANGE key start end        | 获取列表 start 和 end 之间 的元素          |

3. 使用场景为:

   > **信息流展示**
   >
   > - 举例：最新文章、最新动态。
   > - 相关命令：`LPUSH`、`LRANGE`。
   >
   > **消息队列**
   >
   > 通过 `RPUSH/RPOP`或者`LPUSH/LPOP` 实现栈

4. 实现代码：

   ```go
   package main
   
   import (
   	"net/http"
   	"strconv"
   
   	"github.com/gin-gonic/gin"
   	"github.com/go-redis/redis/v8"
   )
   
   var redisClient *redis.Client
   
   func main() {
   	redisClient = redis.NewClient(&redis.Options{
   		Addr:     "localhost:6379",
   		Password: "",
   		DB:       0,
   	})
   
   	r := gin.Default()
   
   	r.POST("/rpush", rPushHandler)
   	r.POST("/lpush", lPushHandler)
   	r.POST("/lset", lSetHandler)
   	r.GET("/lpop", lPopHandler)
   	r.GET("/rpop", rPopHandler)
   	r.GET("/llen", lLenHandler)
   	r.GET("/lrange", lRangeHandler)
   	r.Run(":8080")
   }
   
   // 在指定列表的尾部（右边）添加一个或多个元素
   func rPushHandler(c *gin.Context) {
   	key := c.PostForm("key")
   	values := c.PostFormArray("values")
   
   	strValues := make([]interface{}, len(values))
   	for i, val := range values {
   		strValues[i] = val
   	}
   
   	err := redisClient.RPush(c, key, strValues...).Err()
   	if err != nil {
   		c.JSON(http.StatusInternalServerError, gin.H{
   			"error":   err.Error(),
   			"message": "RPUSH操作失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"message": "RPUSH操作成功",
   	})
   }
   
   // 在指定列表的头部（左边）添加一个或多个元素
   func lPushHandler(c *gin.Context) {
   	key := c.PostForm("key")
   	values := c.PostFormArray("values")
   
   	strValues := make([]interface{}, len(values))
   	for i, val := range values {
   		strValues[i] = val
   	}
   
   	err := redisClient.LPush(c, key, strValues...).Err()
   	if err != nil {
   		c.JSON(http.StatusInternalServerError, gin.H{
   			"error":   err.Error(),
   			"message": "LPUSH操作失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"message": "LPUSH操作成功",
   	})
   }
   
   // 将指定列表索引 index 位置的值设置为 value,相当于修改index这个位置上的值
   func lSetHandler(c *gin.Context) {
   	key := c.PostForm("key")
   	index, _ := strconv.ParseInt(c.PostForm("index"), 10, 64)
   	value := c.PostForm("value")
   
   	err := redisClient.LSet(c, key, index, value).Err()
   	if err != nil {
   		c.JSON(http.StatusInternalServerError, gin.H{
   			"error":   err.Error(),
   			"message": "LSET操作失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"message": "LSET操作成功",
   	})
   }
   
   // 移除并获取指定列表的第一个元素(最左边)
   func lPopHandler(c *gin.Context) {
   	key := c.Query("key")
   
   	value, err := redisClient.LPop(c, key).Result()
   	if err != nil && err != redis.Nil {
   		c.JSON(http.StatusInternalServerError, gin.H{
   			"error":   err.Error(),
   			"message": "LPOP操作失败",
   		})
   		return
   	}
       
   	c.JSON(http.StatusOK, gin.H{
   		"value":   value,
   		"message": "LPOP操作成功",
   	})
   }
   
   // 移除并获取指定列表的最后一个元素(最右边)
   func rPopHandler(c *gin.Context) {
   	key := c.Query("key")
   
   	value, err := redisClient.RPop(c, key).Result()
   	if err != nil && err != redis.Nil {
   		c.JSON(http.StatusInternalServerError, gin.H{
   			"error":   err.Error(),
   			"message": "RPOP操作失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"value":   value,
   		"message": "RPOP操作成功",
   	})
   }
   
   // 获取列表元素数量
   func lLenHandler(c *gin.Context) {
   	key := c.Query("key")
   
   	length, err := redisClient.LLen(c, key).Result()
   	if err != nil && err != redis.Nil { //redis.Nil防止查询的key为不在数据库中的值
   		c.JSON(http.StatusInternalServerError, gin.H{
   			"error":   err.Error(),
   			"message": "LLEN操作失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"listName": key,
   		"length":   length,
   		"message":  "LLEN操作成功",
   	})
   }
   
   // LRANGE key start end
   func lRangeHandler(c *gin.Context) {
   	key := c.Query("key")
   	start, _ := strconv.ParseInt(c.Query("start"), 10, 64)
   	end, _ := strconv.ParseInt(c.Query("end"), 10, 64)
   
   	result, err := redisClient.LRange(c, key, start, end).Result()
   	if err != nil && err != redis.Nil {
   		c.JSON(http.StatusInternalServerError, gin.H{
   			"error":   err.Error(),
   			"message": "LRANGE操作失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"listName": key,
   		"result":   result,
   		"message":  "LRANGE操作成功",
   	})
   }
   
   ```

   

### Hash（哈希）

1. 底层实现为：Dict、ZipList

2. 常用命令有：

   > | 命令                        | 介绍                                                   |
   > | --------------------------- | ------------------------------------------------------ |
   > | HSET key field value        | 设置指定哈希表中指定字段的值                           |
   > | HSETNX key field value      | 只有指定字段不存在时设置指定字段的值                   |
   > | HINCRBY key field increment | 对指定哈希中的指定字段做运算操作（正数为加，负数为减） |
   > | HGET key field              | 获取指定哈希表中指定字段的值                           |
   > | HMGET key field1 field2 ... | 获取指定哈希表中一个或者多个指定字段的值               |
   > | HGETALL key                 | 获取指定哈希表中所有的键值对                           |
   > | HEXISTS key field           | 查看指定哈希表中指定的字段是否存在                     |
   > | HDEL key field1 field2 ...  | 删除一个或多个哈希表字段                               |
   > | HLEN key                    | 获取指定哈希表中字段的数量                             |

3. 使用场景

   > **对象数据存储场景**
   >
   > - 举例：用户信息、商品信息、文章信息、购物车信息。
   > - 相关命令：`HSET` （设置单个字段的值）、`HMSET`（设置多个字段的值）、`HGET`（获取单个字段的值）、`HMGET`（获取多个字段的值）。

4. 实现代码

   ```go
   package main
   
   import (
   	"github.com/gin-gonic/gin"
   	"github.com/go-redis/redis/v8"
   	"net/http"
   	"strconv"
   )
   
   var redisClient *redis.Client
   
   type Data struct {
   	Key   string `form:"key"json:"key"`
   	Field string `from:"field"json:"field"`
   	Value string `from:"value"json:"value"`
   }
   
   func main() {
   	redisClient = redis.NewClient(&redis.Options{
   		Addr:     "localhost:6379",
   		Password: "",
   		DB:       0,
   	})
   	r := gin.Default()
   	r.POST("/hset", hsetHandler)
   	r.POST("/hsetnx", hsetnxHandler)
   	r.GET("/hget", hgetHandler)
   	r.GET("/hmget", hmgetHandler)
   	r.GET("/hgetall", hgetallHandler)
   	r.GET("/hexists", hexistsHandler)
   	r.POST("/hdel", hdelHandler)
   	r.GET("/hlen", hlenHandler)
   	r.POST("/hincrby", hincrbyHandler)
   	r.Run(":8080")
   }
   
   // 设置指定哈希表中指定字段的值
   func hsetHandler(c *gin.Context) {
   	var data Data
   	if err := c.ShouldBindJSON(&data); err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "数据绑定失败",
   		})
   		return
   	}
   
   	err := redisClient.HSet(c, data.Key, data.Field, data.Value).Err() //如何实现string和map类型
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "设置指定哈希表中指定字段的值失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"message": "设置指定哈希表中指定字段的值成功",
   	})
   }
   
   // 只有指定字段不存在时设置指定字段的值
   func hsetnxHandler(c *gin.Context) {
   	var data Data
   	if err := c.ShouldBindJSON(&data); err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "数据绑定失败",
   		})
   		return
   	}
   
   	result, err := redisClient.HSetNX(c, data.Key, data.Field, data.Value).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "只有指定字段不存在时设置指定字段的值失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"message": "hsetnx只有指定字段不存在时设置指定字段的值成功",
   		"result":  result,
   	})
   }
   
   // 获取指定哈希表中指定字段的值
   func hgetHandler(c *gin.Context) {
   	key := c.Query("key")
   	field := c.Query("field")
   
   	value, err := redisClient.HGet(c, key, field).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定哈希表中指定字段的值失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"value":   value,
   		"message": "获取指定哈希表中指定字段的值成功",
   	})
   }
   
   // 获取指定哈希表中一个或者多个指定字段的值.
   func hmgetHandler(c *gin.Context) {
   	key := c.Query("key")
   	fields := c.QueryArray("fields")
   
   	values, err := redisClient.HMGet(c, key, fields...).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定哈希表中一个或者多个指定字段的值失败",
   		})
   		return
   	}
   
   	result := make(map[string]interface{})
   	for i, field := range fields {
   		result[field] = values[i]
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"data":    result,
   		"message": "获取指定哈希表中一个或者多个指定字段的值成功",
   	})
   }
   
   // 获取指定哈希表中所有的键值对
   func hgetallHandler(c *gin.Context) {
   	key := c.Query("key")
   
   	result, err := redisClient.HGetAll(c, key).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定哈希表中所有的键值对失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"data":    result,
   		"message": "获取指定哈希表中所有的键值对成功",
   	})
   }
   
   // 查看指定哈希表中指定的字段是否存在
   func hexistsHandler(c *gin.Context) {
   	key := c.Query("key")
   	field := c.Query("field")
   
   	result, err := redisClient.HExists(c, key, field).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "查看指定哈希表中指定的字段是否存在失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "查看指定哈希表中指定的字段是否存在成功",
   	})
   }
   
   // 删除一个或多个哈希表字段
   func hdelHandler(c *gin.Context) {
   	key := c.PostForm("key")
   	fields := c.PostFormArray("fields")
   
   	count, err := redisClient.HDel(c, key, fields...).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "删除一个或多个哈希表字段失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"count":   count,
   		"message": "删除一个或多个哈希表字段成功",
   	})
   }
   
   // 获取指定哈希表中字段的数量
   func hlenHandler(c *gin.Context) {
   	key := c.Query("key")
   
   	count, err := redisClient.HLen(c, key).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定哈希表中字段的数量失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"count":   count,
   		"message": "获取指定哈希表中字段的数量成功",
   	})
   }
   
   // 对指定哈希中的指定字段做运算操作（正数为加，负数为减）
   func hincrbyHandler(c *gin.Context) {
   	key := c.PostForm("key")
   	field := c.PostForm("field")
   	increment := c.PostForm("increment")
   	inc, _ := strconv.ParseInt(increment, 10, 64)
   	result, err := redisClient.HIncrBy(c, key, field, inc).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "对指定哈希中的指定字段做运算操作失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "对指定哈希中的指定字段做运算操作成功",
   	})
   }
   
   ```

   

### Set

1. 底层实现：Dict、Intset

2. 常用命令：

   > | 命令                                  | 介绍                                      |
   > | ------------------------------------- | ----------------------------------------- |
   > | SADD key member1 member2 ...          | 向指定集合添加一个或多个元素              |
   > | SMEMBERS key                          | 获取指定集合中的所有元素                  |
   > | SCARD key                             | 获取指定集合的元素数量                    |
   > | SISMEMBER key member                  | 判断指定元素是否在指定集合中              |
   > | SINTER key1 key2 ...                  | 获取给定所有集合的交集                    |
   > | SINTERSTORE destination key1 key2 ... | 将给定所有集合的交集存储在 destination 中 |
   > | SUNION key1 key2 ...                  | 获取给定所有集合的并集                    |
   > | SUNIONSTORE destination key1 key2 ... | 将给定所有集合的并集存储在 destination 中 |
   > | SDIFF key1 key2 ...                   | 获取给定所有集合的差集                    |
   > | SDIFFSTORE destination key1 key2 ...  | 将给定所有集合的差集存储在 destination 中 |
   > | SPOP key count                        | 随机移除并获取指定集合中一个或多个元素    |
   > | SRANDMEMBER key count                 | 随机获取指定集合中指定数量的元素          |

3. 使用场景：

   > **需要存放的数据不能重复的场景**
   >
   > - 举例：网站 UV 统计（数据量巨大的场景还是 `HyperLogLog`更适合一些）、文章点赞、动态点赞等场景。
   > - 相关命令：`SCARD`（获取集合数量） 。
   >
   > **需要获取多个数据源交集、并集和差集的场景**
   >
   > - 举例：共同好友(交集)、共同粉丝(交集)、共同关注(交集)、好友推荐（差集）、音乐推荐（差集）、订阅号推荐（差集+交集） 等场景。
   > - 相关命令：`SINTER`（交集）、`SINTERSTORE` （交集）、`SUNION` （并集）、`SUNIONSTORE`（并集）、`SDIFF`（差集）、`SDIFFSTORE` （差集）。
   >
   > **需要随机获取数据源中的元素的场景**
   >
   > - 举例：抽奖系统、随机点名等场景。
   > - 相关命令：`SPOP`（随机获取集合中的元素并移除，适合不允许重复中奖的场景）、`SRANDMEMBER`（随机获取集合中的元素，适合允许重复中奖的场景）

4. 相关代码：

   ```go
   package main
   
   import (
   	"github.com/gin-gonic/gin"
   	"github.com/go-redis/redis/v8"
   	"net/http"
   	"strconv"
   )
   
   var redisClient *redis.Client
   
   type Data struct {
   	Key     string   `form:"key" json:"key"`
   	Members []string `form:"members[]" json:"members"`
   	Field   string   `form:"field" json:"field"`
   	Value   string   `form:"value" json:"value"`
   }
   
   func main() {
   	redisClient = redis.NewClient(&redis.Options{
   		Addr:     "localhost:6379",
   		Password: "",
   		DB:       0,
   	})
   	r := gin.Default()
   	r.POST("/sadd", saddHandler)
   	r.GET("/smembers", smembersHandler)
   	r.GET("/scard", scardHandler)
   	r.GET("/sismember", sismemberHandler)
   	r.GET("/sinter", sinterHandler)
   	r.GET("/sinterstore", sinterstoreHandler)
   	r.GET("/sunion", sunionHandler)
   	r.GET("/sunionstore", sunionstoreHandler)
   	r.GET("/sdiff", sdiffHandler)
   	r.GET("/sdiffstore", sdiffstoreHandler)
   	r.GET("/spop", spopHandler)
   	r.GET("/srandmember", srandmemberHandler)
   	r.Run(":8080")
   }
   
   // 向指定集合添加一个或多个元素
   func saddHandler(c *gin.Context) {
   	var data Data
   	if err := c.ShouldBindJSON(&data); err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "数据绑定失败",
   		})
   		return
   	}
   
   	members := make([]interface{}, len(data.Members))
   	for i, member := range data.Members {
   		members[i] = member
   	}
   
   	result, err := redisClient.SAdd(c, data.Key, members...).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "向指定集合添加元素失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "向指定集合添加元素成功",
   	})
   }
   
   // 获取指定集合中的所有元素
   func smembersHandler(c *gin.Context) {
   	key := c.Query("key")
   
   	result, err := redisClient.SMembers(c, key).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定集合中的所有元素失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "获取指定集合中的所有元素成功",
   	})
   }
   
   // 获取指定集合的元素数量
   func scardHandler(c *gin.Context) {
   	key := c.Query("key")
   
   	count, err := redisClient.SCard(c, key).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定集合的元素数量失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"count":   count,
   		"message": "获取指定集合的元素数量成功",
   	})
   }
   
   // 判断指定元素是否在指定集合中
   func sismemberHandler(c *gin.Context) {
   	key := c.Query("key")
   	member := c.Query("member")
   
   	result, err := redisClient.SIsMember(c, key, member).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "判断指定元素是否在指定集合中失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "判断指定元素是否在指定集合中成功",
   	})
   }
   
   // 获取给定所有集合的交集
   func sinterHandler(c *gin.Context) {
   	keys := c.QueryArray("keys")
   
   	result, err := redisClient.SInter(c, keys...).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取给定所有集合的交集失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "获取给定所有集合的交集成功",
   	})
   }
   
   // 将给定所有集合的交集存储在 destination 中
   func sinterstoreHandler(c *gin.Context) {
   	destination := c.Query("destination")
   	keys := c.QueryArray("keys")
   
   	count, err := redisClient.SInterStore(c, destination, keys...).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "将给定所有集合的交集存储失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"count":   count,
   		"message": "将给定所有集合的交集存储成功",
   	})
   }
   
   // 获取给定所有集合的并集
   func sunionHandler(c *gin.Context) {
   	keys := c.QueryArray("keys")
   
   	result, err := redisClient.SUnion(c, keys...).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取给定所有集合的并集失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "获取给定所有集合的并集成功",
   	})
   }
   
   // 将给定所有集合的并集存储在 destination 中
   func sunionstoreHandler(c *gin.Context) {
   	destination := c.Query("destination")
   	keys := c.QueryArray("keys")
   
   	count, err := redisClient.SUnionStore(c, destination, keys...).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "将给定所有集合的并集存储失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"count":   count,
   		"message": "将给定所有集合的并集存储成功",
   	})
   }
   
   // 获取给定所有集合的差集
   func sdiffHandler(c *gin.Context) {
   	keys := c.QueryArray("keys")
   
   	result, err := redisClient.SDiff(c, keys...).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取给定所有集合的差集失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "获取给定所有集合的差集成功",
   	})
   }
   
   // 将给定所有集合的差集存储在 destination 中
   func sdiffstoreHandler(c *gin.Context) {
   	destination := c.Query("destination")
   	keys := c.QueryArray("keys")
   
   	count, err := redisClient.SDiffStore(c, destination, keys...).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "将给定所有集合的差集存储失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"count":   count,
   		"message": "将给定所有集合的差集存储成功",
   	})
   }
   
   // 随机移除并获取指定集合中一个
   func spopHandler(c *gin.Context) {
   	key := c.Query("key")
   
   	result, err := redisClient.SPop(c, key).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "移除指定集合中的元素失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "移除指定集合中的元素成功",
   	})
   }
   
   // 随机获取指定集合中一个或多个元素
   func srandmemberHandler(c *gin.Context) {
   	key := c.Query("key")
   	count, _ := strconv.Atoi(c.DefaultQuery("count", "1"))
   
   	result, err := redisClient.SRandMemberN(c, key, int64(count)).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "随机获取指定集合中的元素失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "随机获取指定集合中的元素成功",
   	})
   }
   ```

   

### Sorted Set（ZSet）

1. 底层实现：ZipList、SkipList

2. 基本命令：

   > | 命令                                          | 介绍                                                         |
   > | --------------------------------------------- | ------------------------------------------------------------ |
   > | ZADD key score1 member1 score2 member2 ...    | 向指定有序集合添加一个或多个元素                             |
   > | ZCARD KEY                                     | 获取指定有序集合的元素数量                                   |
   > | ZSCORE key member                             | 获取指定有序集合中指定元素的 score 值                        |
   > | ZINTERSTORE destination numkeys key1 key2 ... | 将给定所有有序集合的交集存储在 destination 中，对相同元素对应的 score 值进行 SUM 聚合操作，numkeys 为集合数量 |
   > | ZUNIONSTORE destination numkeys key1 key2 ... | 求并集，其它和 ZINTERSTORE 类似                              |
   > | ZDIFFSTORE destination numkeys key1 key2 ...  | 求差集，其它和 ZINTERSTORE 类似                              |
   > | ZRANGE key start end                          | 获取指定有序集合 start 和 end 之间的元素（score 从低到高）   |
   > | ZREVRANGE key start end                       | 获取指定有序集合 start 和 end 之间的元素（score 从高到底）   |
   > | ZREVRANK key member                           | 获取指定有序集合中指定元素的排名(score 从大到小排序)         |

3. 使用场景：

   > **需要随机获取数据源中的元素根据某个权重进行排序的场景**
   >
   > - 举例：各种排行榜比如直播间送礼物的排行榜、朋友圈的微信步数排行榜、王者荣耀中的段位排行榜、话题热度排行榜等等。
   > - 相关命令：`ZRANGE` (从小到大排序)、 `ZREVRANGE` （从大到小排序）、`ZREVRANK` (指定元素排名)。
   >
   > **需要存储的数据有优先级或者重要程度的场景** 比如优先级任务队列。
   >
   > - 举例：优先级任务队列。
   > - 相关命令：`ZRANGE` (从小到大排序)、 `ZREVRANGE` （从大到小排序）、`ZREVRANK` (指定元素排名)。
   
4. 实现代码：
   ```go
   package main
   
   import (
   	"github.com/gin-gonic/gin"
   	"github.com/go-redis/redis/v8"
   	"net/http"
   	"strconv"
   )
   
   var redisClient *redis.Client
   
   type Data struct {
   	Key    string `form:"key" json:"key"`
   	Score  int64  `form:"score" json:"score"`
   	Member string `form:"member" json:"member"`
   }
   
   func main() {
   	redisClient = redis.NewClient(&redis.Options{
   		Addr:     "localhost:6379",
   		Password: "",
   		DB:       0,
   	})
   
   	r := gin.Default()
   
   	r.POST("/zadd", zaddHandler)
   	r.GET("/zcard", zcardHandler)
   	r.GET("/zscore", zscoreHandler)
   	r.POST("/zinterstore", zinterstoreHandler)
   	r.POST("/zunionstore", zunionstoreHandler)
   	r.POST("/zdiffstore", zdiffstoreHandler)
   	r.GET("/zrange", zrangeHandler)
   	r.GET("/zrevrange", zrevrangeHandler)
   	r.GET("/zrevrank", zrevrankHandler)
   
   	r.Run(":8080")
   }
   
   // 向指定有序集合添加一个或多个元素
   func zaddHandler(c *gin.Context) {
   	var data []Data
   	if err := c.ShouldBindJSON(&data); err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "数据绑定失败",
   		})
   		return
   	}
   
   	z := make([]*redis.Z, len(data))
   	for i, d := range data {
   		z[i] = &redis.Z{
   			Score:  float64(d.Score),
   			Member: d.Member,
   		}
   	}
   
   	ctx := c.Request.Context() // 从 gin 的上下文中获取 context.Context
   	err := redisClient.ZAdd(ctx, data[0].Key, z...).Err()
   	if err != nil {
   		c.JSON(http.StatusInternalServerError, gin.H{
   			"error":   err.Error(),
   			"message": "向指定有序集合添加元素失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"message": "向指定有序集合添加元素成功",
   	})
   	//以下写法会报错，报错内容为：redis: can't marshal *string (implement encoding.BinaryMarshaler)
   	//var data []Data
   	//if err := c.ShouldBindJSON(&data); err != nil {
   	//	c.JSON(http.StatusBadRequest, gin.H{
   	//		"error":   err.Error(),
   	//		"message": "数据绑定失败",
   	//	})
   	//	return
   	//}
   	//
   	//z := make([]*redis.Z, 0)
   	//for _, d := range data {
   	//	//z[i].Member = &d.Member
   	//	//z[i].Score = float64(d.Score)
   	//	z = append(z, &redis.Z{
   	//		Score:  float64(d.Score),
   	//		Member: &d.Member,
   	//	})
   	//}
   	//
   	//err := redisClient.ZAdd(c, data[0].Key, z...).Err()
   	//if err != nil {
   	//	c.JSON(http.StatusBadRequest, gin.H{
   	//		"error":   err.Error(),
   	//		"message": "向指定有序集合添加元素失败",
   	//	})
   	//	return
   	//}
   	//
   	//c.JSON(http.StatusOK, gin.H{
   	//	"message": "向指定有序集合添加元素成功",
   	//})
   }
   
   // 获取指定有序集合的元素数量
   func zcardHandler(c *gin.Context) {
   	key := c.Query("key")
   
   	count, err := redisClient.ZCard(c, key).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定有序集合的元素数量失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"count":   count,
   		"message": "获取指定有序集合的元素数量成功",
   	})
   }
   
   // 获取指定有序集合中指定元素的 score 值
   func zscoreHandler(c *gin.Context) {
   	key := c.Query("key")
   	member := c.Query("member")
   
   	score, err := redisClient.ZScore(c, key, member).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定有序集合中指定元素的 score 值失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"score":   score,
   		"message": "获取指定有序集合中指定元素的 score 值成功",
   	})
   
   }
   
   // 将给定所有有序集合的交集存储在 destination 中，对相同元素对应的 score 值进行 SUM 聚合操作
   func zinterstoreHandler(c *gin.Context) {
   	destination := c.PostForm("destination")
   	_, _ = strconv.Atoi(c.PostForm("numKeys"))
   	keys := c.PostFormArray("keys")
   
   	err := redisClient.ZInterStore(c, destination, &redis.ZStore{
   		Keys:      keys,
   		Aggregate: "SUM",
   	}).Err()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "将给定所有有序集合的交集存储失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"message": "将给定所有有序集合的交集存储成功",
   	})
   }
   
   // 求并集，其它和 ZINTERSTORE 类似
   func zunionstoreHandler(c *gin.Context) {
   	destination := c.PostForm("destination")
   	_, _ = strconv.Atoi(c.PostForm("numkeys"))
   	keys := c.PostFormArray("keys")
   
   	err := redisClient.ZUnionStore(c, destination, &redis.ZStore{
   		Keys:      keys,
   		Aggregate: "SUM",
   	}).Err()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "求并集失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"message": "求并集成功",
   	})
   }
   
   // 求差集，其它和 ZINTERSTORE 类似
   func zdiffstoreHandler(c *gin.Context) {
   	destination := c.PostForm("destination")
   	_, _ = strconv.Atoi(c.PostForm("numkeys"))
   	keys := c.PostFormArray("keys")
   
   	err := redisClient.ZInterStore(c, destination, &redis.ZStore{
   		Keys:      keys,
   		Aggregate: "SUM",
   	}).Err()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "求差集失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"message": "求差集成功",
   	})
   }
   
   // 获取指定有序集合 start 和 end 之间的元素（score 从低到高）
   func zrangeHandler(c *gin.Context) {
   	key := c.Query("key")
   	start, _ := strconv.Atoi(c.Query("start"))
   	end, _ := strconv.Atoi(c.Query("end"))
   
   	result, err := redisClient.ZRange(c, key, int64(start), int64(end)).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定有序集合元素失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "获取指定有序集合元素成功",
   	})
   }
   
   // 获取指定有序集合 start 和 end 之间的元素（score 从高到底）
   func zrevrangeHandler(c *gin.Context) {
   	key := c.Query("key")
   	start, _ := strconv.Atoi(c.Query("start"))
   	end, _ := strconv.Atoi(c.Query("end"))
   
   	result, err := redisClient.ZRevRange(c, key, int64(start), int64(end)).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定有序集合元素失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"result":  result,
   		"message": "获取指定有序集合元素成功",
   	})
   }
   
   // 获取指定有序集合中指定元素的排名（score 从大到小排序）
   func zrevrankHandler(c *gin.Context) {
   	key := c.Query("key")
   	member := c.Query("member")
   
   	rank, err := redisClient.ZRevRank(c, key, member).Result()
   	if err != nil {
   		c.JSON(http.StatusBadRequest, gin.H{
   			"error":   err.Error(),
   			"message": "获取指定有序集合中指定元素的排名失败",
   		})
   		return
   	}
   
   	c.JSON(http.StatusOK, gin.H{
   		"rank":    rank,
   		"message": "获取指定有序集合中指定元素的排名成功",
   	})
   }
   ```

   #### postman测试Api

   ```json
   {
   	"info": {
   		"_postman_id": "e602a0d8-59b7-494e-ada4-3757dd0a124d",
   		"name": "GO",
   		"schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json",
   		"_exporter_id": "20752411"
   	},
   	"item": [
   		{
   			"name": "String",
   			"item": [
   				{
   					"name": "post",
   					"request": {
   						"method": "POST",
   						"header": [
   							{
   								"key": "required",
   								"value": "",
   								"type": "text"
   							}
   						],
   						"body": {
   							"mode": "raw",
   							"raw": "{\r\n    \"key\":\"33\",\r\n    \"value\":\"8898\"\r\n}",
   							"options": {
   								"raw": {
   									"language": "json"
   								}
   							}
   						},
   						"url": {
   							"raw": "http://localhost:8080/set/",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"set",
   								""
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "incr",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/incr/sas",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"incr",
   								"sas"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "exist",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "formdata",
   							"formdata": [
   								{
   									"key": "key",
   									"value": "33",
   									"type": "text"
   								}
   							]
   						},
   						"url": {
   							"raw": "http://localhost:8080/exist",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"exist"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "testincr",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "formdata",
   							"formdata": [
   								{
   									"key": "id",
   									"value": "3",
   									"type": "text"
   								}
   							]
   						},
   						"url": {
   							"raw": "http://localhost:7999/api/v/AddProductView",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "7999",
   							"path": [
   								"api",
   								"v",
   								"AddProductView"
   							],
   							"query": [
   								{
   									"key": "id",
   									"value": "3",
   									"disabled": true
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "MSet",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "raw",
   							"raw": "{\r\n   \"1\":\"hkjhh\",\r\n   \"2\":\"hhkhks\",\r\n   \"3\":\"kjhbas\"\r\n}",
   							"options": {
   								"raw": {
   									"language": "json"
   								}
   							}
   						},
   						"url": {
   							"raw": "http://localhost:8080/mset",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"mset"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "MGet",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "raw",
   							"raw": "[\r\n    \"1\",\r\n    \"2\",\r\n    \"3\"\r\n]",
   							"options": {
   								"raw": {
   									"language": "json"
   								}
   							}
   						},
   						"url": {
   							"raw": "http://localhost:8080/mget",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"mget"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "expire",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "formdata",
   							"formdata": [
   								{
   									"key": "key",
   									"value": "sas",
   									"type": "text"
   								},
   								{
   									"key": "duration",
   									"value": "30h",
   									"type": "text"
   								}
   							]
   						},
   						"url": {
   							"raw": "http://localhost:8080/expire",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"expire"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "get",
   					"protocolProfileBehavior": {
   						"disableBodyPruning": true
   					},
   					"request": {
   						"method": "GET",
   						"header": [],
   						"body": {
   							"mode": "formdata",
   							"formdata": []
   						},
   						"url": {
   							"raw": "http://localhost:8080/get?key=33",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"get"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "33"
   								}
   							]
   						}
   					},
   					"response": []
   				}
   			]
   		},
   		{
   			"name": "List",
   			"item": [
   				{
   					"name": "rPush",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "urlencoded",
   							"urlencoded": [
   								{
   									"key": "key",
   									"value": "mylist",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "2",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "233",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "2345",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "232",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "254",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "5",
   									"type": "text"
   								}
   							]
   						},
   						"url": {
   							"raw": "http://localhost:8080/rpush",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"rpush"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "Lpush",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "urlencoded",
   							"urlencoded": [
   								{
   									"key": "key",
   									"value": "LList",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "89",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "424",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "544",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "25",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "654",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "654",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "87",
   									"type": "text"
   								},
   								{
   									"key": "values",
   									"value": "64",
   									"type": "text"
   								}
   							]
   						},
   						"url": {
   							"raw": "http://localhost:8080/lpush",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"lpush"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "LSet",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "urlencoded",
   							"urlencoded": [
   								{
   									"key": "key",
   									"value": "mylist",
   									"type": "text"
   								},
   								{
   									"key": "index",
   									"value": "3",
   									"type": "text"
   								},
   								{
   									"key": "value",
   									"value": "666",
   									"type": "text"
   								}
   							]
   						},
   						"url": {
   							"raw": "http://localhost:8080/lset",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"lset"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "LPop",
   					"protocolProfileBehavior": {
   						"disableBodyPruning": true
   					},
   					"request": {
   						"method": "GET",
   						"header": [],
   						"body": {
   							"mode": "urlencoded",
   							"urlencoded": [
   								{
   									"key": "key",
   									"value": "mylist",
   									"type": "text"
   								}
   							]
   						},
   						"url": {
   							"raw": "http://localhost:8080/lpop?key=mylist",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"lpop"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "mylist"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "RPop",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/rpop?key=mylist",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"rpop"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "mylist"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "llen",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/llen?key=mylist",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"llen"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "mylist"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "lrange",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/lrange?key=mylist&star=1&end=2",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"lrange"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "mylist"
   								},
   								{
   									"key": "star",
   									"value": "1"
   								},
   								{
   									"key": "end",
   									"value": "2"
   								}
   							]
   						}
   					},
   					"response": []
   				}
   			]
   		},
   		{
   			"name": "Hash",
   			"item": [
   				{
   					"name": "hset",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "raw",
   							"raw": "{\r\n    \"key\":\"myhash\",\r\n    \"field\":\"e\",\r\n    \"value\":\"HA\",\r\n    \"field\":\"nel\",\r\n    \"value\":\"haksk\"\r\n}",
   							"options": {
   								"raw": {
   									"language": "json"
   								}
   							}
   						},
   						"url": {
   							"raw": "http://localhost:8080/hset",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"hset"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "87877",
   									"disabled": true
   								},
   								{
   									"key": "value",
   									"value": "799888",
   									"disabled": true
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "hsetnx",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "raw",
   							"raw": "{\r\n    \"key\":\"myh\",\r\n    \"field\":\"n\",\r\n    \"value\":\"hkjhbs\"\r\n}\r\n",
   							"options": {
   								"raw": {
   									"language": "json"
   								}
   							}
   						},
   						"url": {
   							"raw": "http://localhost:8080/hsetnx",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"hsetnx"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "hget",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/hget?key=myhash&field=n",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"hget"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "myhash"
   								},
   								{
   									"key": "field",
   									"value": "n"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "hmget",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/hmget?key=myhash&fields=",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"hmget"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "myhash"
   								},
   								{
   									"key": "fields",
   									"value": ""
   								},
   								{
   									"key": "fields",
   									"value": "n",
   									"disabled": true
   								},
   								{
   									"key": "fields",
   									"value": "nel",
   									"disabled": true
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "hgetall",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/hgetall?key=myhash",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"hgetall"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "myhash"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "hlen",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/hlen?key=myhash",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"hlen"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "myhash"
   								}
   							]
   						}
   					},
   					"response": []
   				}
   			]
   		},
   		{
   			"name": "Set",
   			"item": [
   				{
   					"name": "sadd",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "raw",
   							"raw": "{\r\n    \"key\":\"set\",\r\n    \"members\":[\"18\",\"78798\",\"7897895\",\"2\",\"1\",\"798987\"]\r\n}",
   							"options": {
   								"raw": {
   									"language": "json"
   								}
   							}
   						},
   						"url": {
   							"raw": "http://localhost:8080/sadd",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"sadd"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "smembers",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/smembers?key=myset",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"smembers"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "myset"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "scard",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/scard?key=myset",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"scard"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "myset"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "sinter",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/sinter?keys=set&keys=myset",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"sinter"
   							],
   							"query": [
   								{
   									"key": "keys",
   									"value": "set"
   								},
   								{
   									"key": "keys",
   									"value": "myset"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "destination",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/sinterstore?destination=inter&keys=set&keys=myset",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"sinterstore"
   							],
   							"query": [
   								{
   									"key": "destination",
   									"value": "inter"
   								},
   								{
   									"key": "keys",
   									"value": "set"
   								},
   								{
   									"key": "keys",
   									"value": "myset"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "sunion",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/sunion?keys=set&keys=myset",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"sunion"
   							],
   							"query": [
   								{
   									"key": "keys",
   									"value": "set"
   								},
   								{
   									"key": "keys",
   									"value": "myset"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "sdiff",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/sdiff?keys=set&keys=myset",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"sdiff"
   							],
   							"query": [
   								{
   									"key": "keys",
   									"value": "set"
   								},
   								{
   									"key": "keys",
   									"value": "myset"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "spop",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/spop?key=myset",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"spop"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "myset"
   								},
   								{
   									"key": "count",
   									"value": "2",
   									"disabled": true
   								}
   							]
   						}
   					},
   					"response": []
   				}
   			]
   		},
   		{
   			"name": "ZSet",
   			"item": [
   				{
   					"name": "Zadd",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "raw",
   							"raw": "[\r\n    {\r\n        \"key\": \"mysetss\",\r\n        \"score\": 87,\r\n        \"member\": \"Item1\"\r\n    },\r\n    {\r\n        \r\n        \"score\": 89,\r\n        \"member\": \"Item2\"\r\n    }\r\n]\r\n",
   							"options": {
   								"raw": {
   									"language": "json"
   								}
   							}
   						},
   						"url": {
   							"raw": "http://localhost:8080/zadd",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"zadd"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "zcard",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/zcard?key=mysetss",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"zcard"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "mysetss"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "zscore",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/zscore?key=mysetss&member=Item1",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"zscore"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "mysetss"
   								},
   								{
   									"key": "member",
   									"value": "Item1"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "zinterstore",
   					"request": {
   						"method": "POST",
   						"header": [],
   						"body": {
   							"mode": "urlencoded",
   							"urlencoded": [
   								{
   									"key": "destination",
   									"value": "unionset",
   									"type": "text"
   								},
   								{
   									"key": "keys",
   									"value": "[\"mysetss\",\"mysetsss\"]",
   									"type": "text"
   								}
   							]
   						},
   						"url": {
   							"raw": "http://localhost:8080/zinterstore",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"zinterstore"
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "zrange",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/zrange?key=mysetss&start=0&end=1",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"zrange"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "mysetss"
   								},
   								{
   									"key": "start",
   									"value": "0"
   								},
   								{
   									"key": "end",
   									"value": "1"
   								}
   							]
   						}
   					},
   					"response": []
   				},
   				{
   					"name": "zrevrank",
   					"request": {
   						"method": "GET",
   						"header": [],
   						"url": {
   							"raw": "http://localhost:8080/zrevrank?key=mysetss&member=Item1",
   							"protocol": "http",
   							"host": [
   								"localhost"
   							],
   							"port": "8080",
   							"path": [
   								"zrevrank"
   							],
   							"query": [
   								{
   									"key": "key",
   									"value": "mysetss"
   								},
   								{
   									"key": "member",
   									"value": "Item1"
   								}
   							]
   						}
   					},
   					"response": []
   				}
   			]
   		}
   	]
   }
   ```

   
