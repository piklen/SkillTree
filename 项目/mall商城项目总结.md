## 总概

**本项目通过使用go语言中gin、gorm、redis、jwt等基本技术实现了商城系统的商品创建、用户创建、生成订单、秒杀等功能。**

## 一、基本功能的实现

### 一、用户方面

#### 1.用户注册

![image.png](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/image.png)

**用户批量注册，流程与用户注册类似,存在以下几点不同：**

1. 数据结构不同，前端数据传来的是一个数组

    ```Go
    type BatchUsersService struct {
    	Users []BatchUserService `json:"users" binding:"required"`
    }
    type BatchUserService struct {
    	NickName string `form:"nick_name" json:"nick_name"`
    	UserName string `form:"user_name" json:"user_name"`
    	Password string `form:"password" json:"password"`
    	Key      string `form:"key" json:"key"` // 前端进行验证
    }
    ```

1. 对密码以及账户金额分别进行加密，并且将传来数组解析出一个用户名集合

    ```Go
        user := make([]model.User, len(users))
    	userNames := make([]string, len(users))
    	for i, v := range users {
    		userNames[i] = v.UserName
    		//进行校验密码
    		if v.Key == "" || len(v.Key) != 16 {
    			code = e.Error
    			return serializer.Response{
    				Status: code,
    				Msg:    e.GetMsg(code),
    				Data:   "密钥长度不足",
    			}
    		}
    
    		//10000  ----->密文存储,对称加密操作
    		util.Encrypt.SetKey(v.Key)
    		user[i] = model.User{
    			UserName: v.UserName,
    			NickName: v.NickName,
    			Status:   model.BatchActive,
    			Avatar:   "avatar.jpeg",
    			Money:    util.Encrypt.AesEncoding("10000"), // 初始金额
    		}
    
    		// 加密密码
    		//前端传入的是明文
    		if err := user[i].BatchSetPassword(v.Password); err != nil {
    			code = e.ErrorFailEncryption
    			return serializer.Response{
    				Status: code,
    				Msg:    e.GetMsg(code),
    			}
    		}
    	}
    ```

1. 进行的是对username切片的批量数据库查询

    ```Go
    func (dao *UserDao) BatchExistOrNotByUserNames(userNames []string) ([]*model.User, bool, error) {
    	var users []*model.User
    	var exists bool
    
    	err := dao.DB.Model(&model.User{}).
    		Where("user_name IN (?)", userNames).
    		Find(&users).Error
    
    	if err != nil {
    		return nil, false, err
    	}
        //将重复的username返回到users中，只需要对users进行长度确定，就可以判断是否有重复内容
    	if len(users) != 0 {
    		return users, true, nil
    	}
    	//for _, user := range users {
    	//	if user == nil {
    	//
    	//	} else {
    	//		exists = true
    	//		break
    	//	}
    	//}
    	return users, exists, nil
    }
    
    ```

#### 2.用户登录

![image.png](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/image%201.png)

#### 3.用户昵称修改

![image.png](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/image%202.png)

#### 4.上传用户头像

1. 从gin中获取用户上传的内容，并且进行数据绑定

    ```Go
    file, fileHeader, _ := c.Request.FormFile("file")
    fileSize := fileHeader.Size
    uploadAvatarService := service.UserService{}
    err := c.ShouldBind(&uploadAvatarService)
    ```

1. 通过上传的token解析用户信息

    ```Go
    chaim, _ := util.ParseToken(c.GetHeader("Authorization"))
    ```

1. 通过用户id从数据库中获取到用户信息

    ```Go
    	userDao := dao.NewUserDao(ctx)
    	user, err = userDao.GetUserById(uId)
    	if err != nil {
    		code = e.Error
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    			Error:  err.Error(),
    		}
    	}
    ```

1. 将用户头像内容上传到本地

    ```Go
    	path, err := UploadAvatarToLocalStatic(file, uId, user.UserName)
    	if err != nil {
    		code = e.ErrorUploadFile
    		return serializer.Response{
    			Status: code,
    			Data:   e.GetMsg(code),
    			Error:  err.Error(),
    		}
    	}
    // UploadAvatarToLocalStatic 上传头像
    func UploadAvatarToLocalStatic(file multipart.File, userId uint, userName string) (filePath string, err error) {
    	bId := strconv.Itoa(int(userId)) //路径拼接
    	basePath := "." + conf.AvatarPath + "user" + bId + "/"
    	if !DirExistOrNot(basePath) {
    		CreateDir(basePath)
    	}
    	avatarPath := basePath + userName + ".jpg"
    	content, err := io.ReadAll(file)
    	if err != nil {
    		return "", err
    	}
    	err = os.WriteFile(avatarPath, content, 0666)
    	if err != nil {
    		return "", err
    	}
    	return "user" + bId + "/" + userName + ".jpg", err
    }
    ```

1. 更新用户信息

    ```Go
    user.Avatar = path
    err = userDao.UpdateUserById(uId, user)
    // UpdateUserById 根据 id 更新用户信息
    func (dao *UserDao) UpdateUserById(uId uint, user *model.User) error {
    	return dao.DB.Model(&model.User{}).Where("id=?", uId).
    		Updates(&user).Error
    }
    ```

#### 5.用户邮箱绑定与确认

1. 首先获取用户相关信息

    ```Go
    chaim, _ := util.ParseToken(c.GetHeader("Authorization"))
    ```

1. 生成修改邮箱的token

    ```Go
    	token, err := util.GenerateEmailToken(uId, service.OperationType, service.Email, service.Password)
    	if err != nil {
    		code = e.ErrorAuthToken
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    			Error:  err.Error(),
    		}
    	}
    ```

1. 加载发送消息的模板，通过OperationType 进行判断e1:绑定邮箱 2：解绑邮箱 3：改密码

    ```Go
    noticeDao := dao.NewNoticeDao(ctx)
    	notice, err = noticeDao.GetNoticeById(service.OperationType)
    	if err != nil {
    		code = e.Error
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    			Error:  err.Error(),
    		}
    	}
    ```

1. 构造发送请求

    ```Go
        address = conf.ValidEmail + token //发送方
    	mailStr := notice.Text
    	mailText := strings.Replace(mailStr, "Email", address, -1) //字符串替换
    	m := mail.NewMessage()
    	m.SetHeader("From", conf.SmtpEmail)
    	m.SetHeader("To", service.Email)
    	m.SetHeader("Subject", "xiaobao")
    	m.SetBody("text/html", mailText)
    	d := mail.NewDialer(conf.SmtpHost, 465, conf.SmtpEmail, conf.SmtpPass)
    	d.StartTLSPolicy = mail.MandatoryStartTLS
    	if err := d.DialAndSend(m); err != nil {
    		code = e.ErrorSendEmail
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    		}
    	}
    	return serializer.Response{
    		Status: code,
    		Msg:    e.GetMsg(code),
    	}
    ```

1. 验证token

    ```Go
    	// 验证token
    	if token == "" {
    		code = e.InvalidParams
    	} else {
    		claims, err := util.ParseEmailToken(token)
    		if err != nil {
    			//如果解析token错误就返回错误
    			code = e.ErrorAuthToken
    		} else if time.Now().Unix() > claims.ExpiresAt {
    			//如果超时就返回验证时间超时
    			code = e.ErrorAuthCheckTokenTimeout
    		} else {
    			//不然就是成功了，就直接构建用户结构体
    			userID = claims.UserID
    			email = claims.Email
    			password = claims.Password
    			operationType = claims.OperationType
    		}
    	}
    ```

1. 通过用户id从数据库中获取信息

    ```Go
    	// 获取该用户信息
    	userDao := dao.NewUserDao(ctx)
    	user, err := userDao.GetUserById(userID)
    	if err != nil {
    		code = e.Error
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    		}
    	}
    ```

1. 修改结构体内容，并根据操作进行数据修改

    ```Go
    if operationType == 1 {
    		// 1:绑定邮箱
    		user.Email = email
    	} else if operationType == 2 {
    		// 2：解绑邮箱
    		user.Email = ""
    	} else if operationType == 3 {
    		// 3：修改密码
    		err = user.SetPassword(password)
    		if err != nil {
    			code = e.Error
    			return serializer.Response{
    				Status: code,
    				Msg:    e.GetMsg(code),
    			}
    		}
    	}
    	err = userDao.UpdateUserById(userID, user)
    ```



#### 6.获取用户金额

1. 获取用户id

    ```Go
    claim, _ := util.ParseToken(c.GetHeader("Authorization"))
    ```

1. 再获取用户信息

    ```Go
    	user, err := userDao.GetUserById(uId)
    	if err != nil {
    		code = e.Error
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    			Error:  err.Error(),
    		}
    	}
    ```

1. 再解析用户信息里面的金额密钥

    ```Go
    func BuildMoney(item *model.User, key string) Money {
    	util.Encrypt.SetKey(key)
    	return Money{
    		UserID:    item.ID,
    		UserName:  item.UserName,
    		UserMoney: util.Encrypt.AesDecoding(item.Money),
    	}
    }
    ```

### 二、商品部分

#### 1.商品创建

1. 首先进行路由

    ```Go
    authed.POST("create", api.CreateProduct)
    ```

1. 从context中获取商品图片并且获取用户username等信息

    ```Go
    form, _ := c.MultipartForm()
    files := form.File["file"]
    claim, _ := util.ParseToken(c.GetHeader("Authorization"))
    ```

1. 进行数据绑定

    ```Go
    createProductService := service.ProductService{}
    err := c.ShouldBind(&createProductService)
    ```

1. 获取第一张图片为封面图

    ```Go
    tmp, _ := files[0].Open()
    path, err := UploadProductToLocalStatic(tmp, uId, service.Name)
    func UploadProductToLocalStatic(file multipart.File, bossId uint, productName string) (filePath string, err error) {
    	bId := strconv.Itoa(int(bossId))
    	basePath := "." + conf.ProductPath + "boss" + bId + "/"
    	if !DirExistOrNot(basePath) {
    		CreateDir(basePath)
    	}
    	productPath := basePath + productName + ".jpg"
    	content, err := io.ReadAll(file)
    	if err != nil {
    		return "", err
    	}
    	err = os.WriteFile(productPath, content, 0666)
    	if err != nil {
    		return "", err
    	}
    	return "boss" + bId + "/" + productName + ".jpg", err
    }
    // DirExistOrNot 判断文件夹路径是否存在
    func DirExistOrNot(fileAddr string) bool {
    	s, err := os.Stat(fileAddr)
    	if err != nil {
    		return false
    	}
    	return s.IsDir()
    }
    
    // CreateDir 创建文件夹
    func CreateDir(dirName string) bool {
    	err := os.MkdirAll(dirName, 7550)
    	if err != nil {
    		return false
    	}
    	return true
    }
    ```

1. 构建商品数据结构体并写入数据库

    ```Go
    product := &model.Product{
    		Name:          service.Name,
    		CategoryID:    uint(service.CategoryID),
    		Title:         service.Title,
    		Info:          service.Info,
    		ImgPath:       path,
    		Price:         service.Price,
    		DiscountPrice: service.DiscountPrice,
    		Num:           service.Num,
    		OnSale:        true,
    		BossID:        uId,
    		BossName:      boss.UserName,
    		BossAvatar:    boss.Avatar,
    	}
    	productDao := dao.NewProductDao(ctx)
    	err = productDao.CreateProduct(product)
    ```

1. 通过并发协调将图片写入数据库，当做商品详情介绍

    ```Go
    	wg := new(sync.WaitGroup)
    	wg.Add(len(files))
    	for index, file := range files {
    		go func() {
    			num := strconv.Itoa(index)
    			productImgDao := dao.NewProductImgDaoByDB(productDao.DB)
    			tmp, _ = file.Open()
    			path, err := UploadProductToLocalStatic(tmp, uId, service.Name+num)
    			if err != nil {
    				code = e.ErrorUploadFile
    				panic(err)
    			}
    			productImg := &model.ProductImg{
    				ProductID: product.ID,
    				ImgPath:   path,
    			}
    			err = productImgDao.CreateProductImg(productImg)
    			if err != nil {
    				code = e.Error
    				panic(err)
    			}
    		}()
    		wg.Done()
    	}
    
    	wg.Wait()
    ```

注意点：用并发协调`wg := new(sync.WaitGroup)`来保证所有图片都能够写入数据库

#### 2.商品搜索

1. 首先进行路由

    ```Go
    v.POST("products/search", api.SearchProducts)
    ```

1. 进行数据的绑定

    ```Go
    searchProductsService := service.ProductService{}
    err := c.ShouldBind(&searchProductsService)
    ```

1. 通过模糊匹配进行数据库查询

    ```Go
    func (dao *ProductDao) SearchProduct(info string, pageNum int, pageSize int) (products []*model.Product, err error) {
    	err = dao.DB.Model(&model.Product{}).
    		Where("name LIKE ? OR info LIKE ?", "%"+info+"%", "%"+info+"%").
    		Offset((pageNum - 1) * pageSize).
    		Limit(pageSize).Find(&products).Error
    	return
    }
    ```

1. 进行数据序列化

    ```Go
    // 序列化商品
    func BuildProduct(item *model.Product) Product {
    	p := Product{
    		ID:            item.ID,
    		Name:          item.Name,
    		CategoryID:    item.CategoryID,
    		Title:         item.Title,
    		Info:          item.Info,
    		ImgPath:       conf.PhotoHost + conf.HttpPort + conf.ProductPath + item.ImgPath,
    		Price:         item.Price,
    		DiscountPrice: item.DiscountPrice,
    		View:          item.View(),
    		Num:           item.Num,
    		OnSale:        item.OnSale,
    		CreatedAt:     item.CreatedAt.Unix(),
    		BossID:        int(item.BossID),
    		BossName:      item.BossName,
    		BossAvatar:    conf.PhotoHost + conf.HttpPort + conf.AvatarPath + item.BossAvatar,
    	}
    
    	if conf.UploadModel == "oss" {
    		p.ImgPath = item.ImgPath
    		p.BossAvatar = item.BossAvatar
    	}
    
    	return p
    
    }
    
    // 序列化商品列表
    func BuildProducts(items []*model.Product) (products []Product) {
    	for _, item := range items {
    		product := BuildProduct(item)
    		products = append(products, product)
    	}
    	return products
    }
    
    ```

1. 获取点击次数中使用到了Redis

    ```Go
    // View 获取点击数
    func (product *Product) View() uint64 {
    	countStr, _ := cache.RedisClient.Get(cache.ProductViewKey(product.ID)).Result()
    	count, _ := strconv.ParseUint(countStr, 10, 64)
    	return count
    }
    ```



#### 3.通过商品id查询商品信息、通过商品id获取商品图片逻辑是相似的

关键点：返回的内容是一个切片数组，序列化的时候需要对切片内的每一个数组元素进行序列化

#### 4.通过查询进行全部商品展示，或者查询某一类商品

1. 判断是查询全部商品还是说只查询某一类商品

    ```Go
    	//找某种分类商品还是说查找全部商品
    	condition := make(map[string]interface{})
    	if service.CategoryID != 0 {
    		condition["category_id"] = service.CategoryID
    	}
    ```

1. 通过并发协调函数`new(sync.WaitGroup)`来提高查询的效率

    ```Go
    	wg := new(sync.WaitGroup)
    	wg.Add(1)
    	go func() {
    		productDao = dao.NewProductDaoByDB(productDao.DB)
    		products, _ = productDao.ListProductByCondition(condition, service.BasePage)
    		wg.Done()
    	}()
    	wg.Wait()
    ```

#### 5.查找商品类目

就是使用Gin，以及gorm对一个数据库进行查询全部内容

```Go
func (dao *CategoryDao) ListCategory() (category []*model.Category, err error) {
	err = dao.DB.Model(&model.Category{}).Find(&category).Error
	return
}
```

#### 6.获取商品点击量，以及增加商品点击量

1. 先判断id是否可以正常的转化为整数类型

    ```Go
    		id, err := strconv.Atoi(strconv.Itoa(product.ID))
    		if err != nil {
    			c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid product ID"})
    			return
    		}
    ```

1. 构建结构体，并调用相关函数

    ```Go
    		product := &Product{ID: int(id)}
    		count := product.View()		
            product := &Product{ID: int(id)}
    		product.AddView(
    ```

1. Redis实现对点击量的查询与变化

    ```Go
    // View 获取点击数
    func (product *Product) View() uint64 {
    	countStr, _ := cache.RedisClient.Get(cache.ProductViewKey(uint(product.ID))).Result()
    	count, _ := strconv.ParseUint(countStr, 10, 64)
    	return count
    }
    
    // AddView 商品游览
    func (product *Product) AddView() {
    	// 增加视频点击数
    	cache.RedisClient.Incr(cache.ProductViewKey(uint(product.ID)))
    	// 增加排行点击数
    	cache.RedisClient.ZIncrBy(cache.RankKey, 1, strconv.Itoa(int(product.ID)))
    }
    ```

### 三、收藏夹部分

#### 1.收藏夹创建

1. 解析用户信息以及绑定相关数据

    ```Go
    	service := service.FavoritesService{}
    	claim, _ := util.ParseToken(c.GetHeader("Authorization"))
    ```

1. 判断当前商品是否已经在收藏夹里面

    ```Go
    	exist, _ := favoriteDao.FavoriteExistOrNot(service.FavoriteId, uId)
    	if exist {
    		code = e.ErrorExistFavorite
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    		}
    	}
    func (dao *FavoritesDao) FavoriteExistOrNot(pId, uId uint) (exist bool, err error) {
    	var count int64
    	err = dao.DB.Model(&model.Favorite{}).
    		Where("product_id=? And user_id=?", pId, uId).
    		Count(&count).Error
    	if count == 0 && err == nil {
    		return false, err
    	}
    	return true, err
    }
    ```

1. 创建user信息以及创建boss信息、获取商品信息

    ```Go
    	userDao := dao.NewUserDao(ctx)
    	user, err := userDao.GetUserById(uId)
    	//创建BossDao
    	//bossDao需要使用与userDao相同的数据库连接。
    	// 由于NewUserDaoByDB接受一个*gorm.DB参数，可以通过将userDao.DB传递给NewUserDaoByDB来创建一个新的bossDao对象。
    	//这样做是为了确保bossDao使用的是与userDao相同的数据库连接，以便在执行查询操作时保持一致性。
    	bossDao := dao.NewUserDaoByDB(userDao.DB)
    	boss, err := bossDao.GetUserById(service.BossId)
    	if err != nil {
    		code = e.ErrorDatabase
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    		}
    	}
    	//创建ProductDao
    	productDao := dao.NewProductDao(ctx)
    	product, err := productDao.GetProductById(service.ProductId)
    	if err != nil {
    		code = e.ErrorDatabase
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    		}
    	}
    ```

1. 创建收藏夹信息并且进行写入数据库

    ```Go
    	//创建FavoriteDao
    	favorite := &model.Favorite{
    		UserId:    uId,
    		User:      *user,
    		Product:   *product,
    		ProductId: service.ProductId,
    		Boss:      *boss,
    		BossId:    service.BossId,
    	}
    	//进行数据插入
    	favoriteDao = dao.NewFavoritesByDB(favoriteDao.DB)
    	err = favoriteDao.CreateFavorite(favorite)
    	if err != nil {
    		code = e.ErrorDatabase
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    		}
    	}
    ```



#### 2.收藏夹的查询

1. 构建结构体，解析用户信息，进行数据绑定

    ```Go
    service := service.FavoritesService{}
    claim, _ := util.ParseToken(c.GetHeader("Authorization"))
    err := c.ShouldBind(&service)
    ```

1. 通过用户id,对数据库进行查询，并且返回所需要的查询条目数

    ```Go
    favoritesDao := dao.NewFavoritesDao(ctx)
    	if service.PageSize == 0 {
    		service.PageSize = 15
    	}
    favorites, total, err := favoritesDao.ListFavoriteByUserId(uId, service.PageSize, service.PageNum)
    func (dao *FavoritesDao) ListFavoriteByUserId(uId uint, pageSize, pageNum int) (favorites []*model.Favorite, total int64, err error) {
    	err = dao.DB.Model(&model.Favorite{}).Preload("User").
    		Where("user_id=?", uId).Count(&total).Error
    	if err != nil {
    		return
    	}
    	err = dao.DB.Model(model.Favorite{}).Preload("User").Where("user_id=?", uId).
    		Offset((pageNum - 1) * pageSize).
    		Limit(pageSize).Find(&favorites).Error
    	return
    }
    ```

    在Gorm框架中，`.Preload("User")` 是用于预加载关联模型（也称为 Eager Loading）的方法。这个方法的作用是在查询某个模型的同时，预先加载与之关联的其他模型的数据，以避免在后续的代码中多次查询数据库。

    预加载可以提高性能，因为它避免了在后续的代码中多次查询数据库，而是在一次查询中一并获取所有相关联的数据。这在处理一对多或多对多关系时特别有用，以避免 N+1 查询问题，即在加载主模型的同时，需要额外执行 N 次查询来获取关联模型的数据。

1. 序列化返回数据

    ```Go
    // 序列化收藏夹
    func BuildFavorite(item1 *model.Favorite, item2 *model.Product, item3 *model.User) Favorite {
    	return Favorite{
    		UserID:        item1.UserId,
    		ProductID:     item1.ProductId,
    		CreatedAt:     item1.CreatedAt.Unix(),
    		Name:          item2.Name,
    		CategoryID:    item2.CategoryID,
    		Title:         item2.Title,
    		Info:          item2.Info,
    		ImgPath:       item2.ImgPath,
    		Price:         item2.Price,
    		DiscountPrice: item2.DiscountPrice,
    		Num:           item2.Num,
    		OnSale:        item2.OnSale,
    		BossID:        item3.ID,
    	}
    }
    
    // 收藏夹列表
    func BuildFavorites(ctx context.Context, items []*model.Favorite) (favorites []Favorite) {
    	productDao := dao.NewProductDao(ctx)
    	bossDao := dao.NewUserDao(ctx)
    	for _, fav := range items {
    		product, err := productDao.GetProductById(fav.ProductId)
    		if err != nil {
    			continue
    		}
    		boss, err := bossDao.GetUserById(fav.UserId)
    		if err != nil {
    			continue
    		}
    		favorite := BuildFavorite(fav, product, boss)
    		favorites = append(favorites, favorite)
    	}
    	return favorites
    }
    ```



#### 3.收藏夹的删除

1. 先进行数据绑定

    ```Go
    service := service.FavoritesService{}
    err := c.ShouldBind(&service)
    ```

1. 再从绑定的数据中拿到商品id,并且通过这个商品id对商品进行删除

    ```Go
    	favoriteDao := dao.NewFavoritesDao(ctx)
    	err := favoriteDao.DeleteFavoriteById(service.FavoriteId)
    // DeleteFavoriteById 删除收藏夹
    func (dao *FavoritesDao) DeleteFavoriteById(fId uint) error {
    	return dao.DB.Where("id=?", fId).Delete(&model.Favorite{}).Error
    }
    ```

### 四、地址模块

#### 1.用户地址创建

1. 先进行数据绑定以及将用户信息通过token进行解析

    ```Go
    addressService := service.AddressService{}
    claim, _ := util.ParseToken(c.GetHeader("Authorization"))
    err := c.ShouldBind(&addressService)
    ```

1. 将地址数据进行写入数据库

    ```Go
    	addressDao := dao.NewAddressDao(ctx)
    	address := &model.Address{
    		UserID:  uId,
    		Name:    service.Name,
    		Phone:   service.Phone,
    		Address: service.Address, //并没有进行校验
    	}
    	err := addressDao.CreateAddress(address)
    // CreateAddress 创建地址
    func (dao *AddressDao) CreateAddress(address *model.Address) (err error) {
    	err = dao.DB.Model(&model.Address{}).Create(&address).Error
    	return
    }
    ```

1. 将全部地址信息通过用户id进行查询数据库并且进行返回

    ```Go
    	addressDao = dao.NewAddressDaoByDB(addressDao.DB)
    	var addresses []*model.Address
    	addresses, err = addressDao.ListAddressByUid(uId)
    	if err != nil {
    		logging.Info(err)
    		code = e.ErrorDatabase
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    			Error:  err.Error(),
    		}
    	}
    // ListAddressByUid 根据 User Id 获取address
    func (dao *AddressDao) ListAddressByUid(uid uint) (addressList []*model.Address, err error) {
    	err = dao.DB.Model(&model.Address{}).
    		Where("user_id=?", uid).Order("created_at desc").
    		Find(&addressList).Error
    	return
    }
    ```



#### 2.查询用户全部地址跟查询用户某一地址id的地址

不同点在与一个是传入地址id，另一个是对地址数据库查询用户id

```Go
res := addressService.Show(c.Request.Context(), c.Query("id"))
```

```Go
address, err := addressDao.ListAddressByUid(uId)
```



#### 3.用户地址修改

1. 获取用户信息

    ```Go
    addressService := service.AddressService{}
    claim, _ := util.ParseToken(c.GetHeader("Authorization"))
    err := c.ShouldBind(&addressService)
    res := addressService.Update(c.Request.Context(), claim.ID, c.Param("id"))
    ```

1. 生产新的该地址id的地址，并且写入数据库

    ```Go
    	addressDao := dao.NewAddressDao(ctx)
    	address := &model.Address{
    		UserID:  uid,
    		Name:    service.Name,
    		Phone:   service.Phone,
    		Address: service.Address,
    	}
    	addressId, _ := strconv.Atoi(aid)
    	err := addressDao.UpdateAddressById(uint(addressId), address)
    // UpdateAddressById 通过 id 修改地址信息
    func (dao *AddressDao) UpdateAddressById(aId uint, address *model.Address) (err error) {
    	err = dao.DB.Model(&model.Address{}).
    		Where("id=?", aId).Updates(address).Error
    	return
    }
    ```

1. 序列返回当前用户的全部地址

    ```Go
    	addressDao = dao.NewAddressDaoByDB(addressDao.DB)
    	var addresses []*model.Address
    	addresses, err = addressDao.ListAddressByUid(uid)
    ```



#### 4.删除某一地址（其实我感觉是不完善的，他没有进行用户信息鉴权）

1. 获取某一地址的id

    ```Go
    res := addressService.Delete(c.Request.Context(), c.Param("id"))
    ```

1. 通过该地址的id进行删除

    ```Go
    err := addressDao.DeleteAddressById(uint(addressId))
    // DeleteAddressById 根据 id 删除地址
    func (dao *AddressDao) DeleteAddressById(aId uint) (err error) {
    	err = dao.DB.Where("id=?", aId).Delete(&model.Address{}).Error
    	return
    }
    ```

### 五、购物车模块

#### 1.购物车创建

1. 解析用户信息，并且绑定数据

    ```Go
    	createCartService := service.CartService{}
    	claim, _ := util.ParseToken(c.GetHeader("Authorization"))
        err := c.ShouldBind(&createCartService)
    ```

1. 判断当前商品是否在商品数据库中是否存在

    ```Go
    	// 判断商品数据库有无这个商品
    	productDao := dao.NewProductDao(ctx)
    	product, err := productDao.GetProductById(service.ProductId)
    	if err != nil {
    		logging.Info(err)
    		code = e.ErrorDatabase
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    			Error:  err.Error(),
    		}
    	}
    ```

1. 对购物车数据库进行处理

    ```Go
    	cart, err = dao.GetCartById(pId, uId, bId)
    	// 空的，第一次加入
    	if err == gorm.ErrRecordNotFound {
    		cart = &model.Cart{
    			UserId:    uId,
    			ProductId: pId,
    			BossId:    bId,
    			Num:       1,
    			MaxNum:    10,
    			Check:     false,
    		}
    		err = dao.DB.Create(&cart).Error
    		if err != nil {
    			return
    		}
    		return cart, e.Success, err
    	} else if cart.Num < cart.MaxNum {
    		// 小于最大 num
    		cart.Num++
    		err = dao.DB.Save(&cart).Error
    		if err != nil {
    			return
    		}
    		return cart, e.ErrorProductExistCart, err
    	} else {
    		// 大于最大num
    		return cart, e.ErrorProductMoreCart, err
    	}
    ```

#### 2.购物车内容查询

- 通过user_id对购物车进行查询

#### 3.购物车数量修改

- 解析购物车商品的id,并且修改数据库中的商品数量

    ```Go
    res := updateCartService.Update(c.Request.Context(), c.Param("id"))
    ```

    ```Go
    cartDao := dao.NewCartDao(ctx)
    err := cartDao.UpdateCartNumById(uint(cartId), service.Num)
    ```

    ```Go
    // UpdateCartNumById 通过id更新Cart信息
    func (dao *CartDao) UpdateCartNumById(cId, num uint) error {
    	return dao.DB.Model(&model.Cart{}).
    		Where("id=?", cId).Update("num", num).Error
    }
    ```

#### 4.购物车的删除

- 通过id删除购物车内容

    ```Go
    cartDao := dao.NewCartDao(ctx)
    err := cartDao.DeleteCartById(service.Id)
    // DeleteCartById 通过 cart_id 删除 cart
    func (dao *CartDao) DeleteCartById(cId uint) error {
    	return dao.DB.Model(&model.Cart{}).
    		Where("id=?", cId).Delete(&model.Cart{}).Error
    }
    ```

### 六、订单模块

#### 1.订单的创建

1. 创建订单结构体、绑定数据、解析用户信息

    ```Go
    	service := service.OrderService{}
    	claim, _ := util.ParseToken(c.GetHeader("Authorization"))
        err := c.ShouldBind(&service)
    ```

1. 构建订单基本结构

    ```Go
    //构造order数据
    	order := &model.Order{
    		UserId:    id,
    		ProductId: service.ProductID,
    		BossId:    service.BossID,
    		Num:       int(service.Num),
    		Money:     float64(service.Money),
    		Type:      1,
    	}
    ```

1. 获取地址信息

    ```Go
    	//构造address地址通过AddressId
    	addressDao := dao.NewAddressDao(ctx)
    	address, err := addressDao.GetAddressByAid(service.AddressID)
    	if err != nil {
    		logging.Info(err)
    		code = e.ErrorDatabase
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    			Error:  err.Error(),
    		}
    	}
    ```

1. 生产订单号

    ```Go
    	//给订单生成唯一订单号，订单号由随机数字字符串、产品ID字符串和用户ID字符串连接起来
    	order.AddressId = address.ID
    	number := fmt.Sprintf("%09v", rand.New(rand.NewSource(time.Now().UnixNano())).Int31n(1000000000))
    	productNum := strconv.Itoa(int(service.ProductID))
    	userNum := strconv.Itoa(int(id))
    	number = number + productNum + userNum
    	orderNum, _ := strconv.ParseUint(number, 10, 64)
    	order.OrderNum = orderNum
    
    ```

1. 将订单进行插入数据库

    ```Go
    	//订单进行插入数据库操作
    	orderDao := dao.NewOrderDao(ctx)
    	err = orderDao.CreateOrder(order)
    	if err != nil {
    		logging.Info(err)
    		code = e.ErrorDatabase
    		return serializer.Response{
    			Status: code,
    			Msg:    e.GetMsg(code),
    			Error:  err.Error(),
    		}
    	}
    ```

1. 将订单存入Redis中，并且设置过期时间

    ```Go
    	// 订单号存入Redis中，设置过期时间
    	data := redis.Z{
    		Score:  float64(time.Now().Unix()) + 15*time.Minute.Seconds(),
    		Member: orderNum,
    	}
    	cache.RedisClient.ZAdd(OrderTimeKey, data)
    ```

#### 2.订单的查询

1. 创建订单结构体、绑定数据、解析用户信息

    ```Go
    service := service.OrderService{}
    claim, _ := util.ParseToken(c.GetHeader("Authorization"))
    err := c.ShouldBind(&service
    ```

1. 设置查询为是否支付或者全部进行查询

    ```Go
    	if service.Type == 0 {
    		condition["type"] = 0
    	} else {
    		condition["type"] = service.Type
    	}
    // ListOrderByCondition 获取订单List
    func (dao *OrderDao) ListOrderByCondition(condition map[string]interface{}, page model.BasePage) (orders []*model.Order, total int64, err error) {
    	err = dao.DB.Model(&model.Order{}).Where(condition).
    		Count(&total).Error
    	if err != nil {
    		return nil, 0, err
    	}
    
    	err = dao.DB.Model(&model.Order{}).Where(condition).
    		Offset((page.PageNum - 1) * page.PageSize).
    		Limit(page.PageSize).Order("created_at desc").Find(&orders).Error
    	return
    }
    ```

#### 3.某一订单的查询

- 通过某一订单的订单id进行查询

    ```Go
        res := showOrderService.Show(c.Request.Context(), c.Query("id"))
    	orderId, _ := strconv.Atoi(uId)
    	orderDao := dao.NewOrderDao(ctx)
    	order, _ := orderDao.GetOrderById(uint(orderId))
    ```

#### 4.订单的删除

- 通过某一订单id进行删除

    ```Go
        res := deleteOrderService.Delete(c.Request.Context(), c.Query("id"))
    	orderDao := dao.NewOrderDao(ctx)
    	orderId, _ := strconv.Atoi(oId)
    	err := orderDao.DeleteOrderById(uint(orderId))
    // DeleteOrderById 获取订单详情
    func (dao *OrderDao) DeleteOrderById(id uint) error {
    	return dao.DB.Where("id=?", id).Delete(&model.Order{}).Error
    }
    
    ```

#### 5.订单的支付

1. 构建结构体、进行数据绑定、解析用户信息

    ```Go
    orderPay := service.OrderPay{}
    claim, _ := util.ParseToken(c.GetHeader("Authorization"))
    res := orderPay.PayDown(c.Request.Context(), claim.ID)
    ```

1. 使用Transaction方法开始一个数据库事务，保证数据一致性

    ```Go
    err := dao.NewOrderDao(ctx).Transaction(func(tx *gorm.DB)
    ```

1. 设置加密密钥

    ```Go
    	//设置加密密钥
    		util.Encrypt.SetKey(o.Key)
    ```

1. 获取订单总金额

    ```Go
    		orderDao := dao.NewOrderByDB(tx)
    		//获取订单总金额，
    		order, err := orderDao.GetOrderById(o.OrderId)
    		if err != nil {
    			logging.Info(err)
    			return err
    		}
    		money := order.Money
    		num := order.Num
    		money = money * float64(num)
    
    		userDao := dao.NewUserDaoByDB(tx)
    		user, err := userDao.GetUserById(uId)
    		if err != nil {
    			logging.Info(err)
    			code = e.ErrorDatabase
    			return err
    		}
    
    ```

1. 对金额进行解密

    ```Go
    		// 对钱进行解密。减去订单。再进行加密。
    		moneyStr := util.Encrypt.AesDecoding(user.Money)
    		moneyFloat, _ := strconv.ParseFloat(moneyStr, 64)
    		if moneyFloat-money < 0.0 { // 金额不足进行回滚
    			logging.Info(err)
    			code = e.ErrorDatabase
    			return errors.New("金币不足")
    		}
    ```

5. 更新用户金额

```Go
		finMoney := fmt.Sprintf("%f", moneyFloat-money)
		user.Money = util.Encrypt.AesEncoding(finMoney)
		//更新用户金额
		err = userDao.UpdateUserById(uId, user)
		if err != nil { // 更新用户金额失败，回滚
			logging.Info(err)
			code = e.ErrorDatabase
			return err
		}
```

1. 更新老板金额

    ```Go
    		//更新boss金币数量
    		boss := new(model.User)
    		boss, err = userDao.GetUserById(uint(o.BossID))
    		moneyStr = util.Encrypt.AesDecoding(boss.Money)
    		moneyFloat, _ = strconv.ParseFloat(moneyStr, 64)
    		finMoney = fmt.Sprintf("%f", moneyFloat+money)
    		boss.Money = util.Encrypt.AesEncoding(finMoney)
    
    		err = userDao.UpdateUserById(uint(o.BossID), boss)
    		if err != nil { // 更新boss金额失败，回滚
    			logging.Info(err)
    			code = e.ErrorDatabase
    			return err
    		}
    ```

1. 更新商品数量

    ```Go
    		//更新商品数量
    		product := new(model.Product)
    		productDao := dao.NewProductDaoByDB(tx)
    		product, err = productDao.GetProductById(uint(o.ProductID))
    		if err != nil {
    			return err
    		}
    		product.Num -= num
    		err = productDao.UpdateProduct(uint(o.ProductID), product)
    		if err != nil { // 更新商品数量减少失败，回滚
    			logging.Info(err)
    			code = e.ErrorDatabase
    			return err
    		}
    ```

1. 更新订单状态

    ```Go
    		// 更新订单状态
    		order.Type = 2
    		err = orderDao.UpdateOrderById(o.OrderId, order)
    		if err != nil { // 更新订单失败，回滚
    			logging.Info(err)
    			code = e.ErrorDatabase
    			return err
    		}
    // UpdateOrderById 更新订单详情
    func (dao *OrderDao) UpdateOrderById(id uint, order *model.Order) error {
    	return dao.DB.Where("id=?", id).Updates(order).Error
    }
    ```

1. 更新商品状态

    ```Go
    		productUser := model.Product{
    			Name:          product.Name,
    			CategoryID:    product.CategoryID,
    			Title:         product.Title,
    			Info:          product.Info,
    			ImgPath:       product.ImgPath,
    			Price:         product.Price,
    			DiscountPrice: product.DiscountPrice,
    			Num:           num,
    			OnSale:        false,
    			BossID:        uId,
    			BossName:      user.UserName,
    			BossAvatar:    user.Avatar,
    		}
    
    		err = productDao.CreateProduct(&productUser)
    		if err != nil { // 买完商品后创建成了自己的商品失败。订单失败，回滚
    			logging.Info(err)
    			code = e.ErrorDatabase
    			return err
    		}
    // CreateProduct 创建商品
    func (dao *ProductDao) CreateProduct(product *model.Product) error {
    	return dao.DB.Model(&model.Product{}).Create(&product).Error
    }
    
    ```



## 二、项目用了哪些技术？

- go基础语法，go并发，map,切片，并发协调

- gin

- gorm

- MySQL

- redis

- jwt、http、https

- cors

## 三、项目在实现过程中遇上了什么问题？

### 1.postman该如何正确的发送请求？

    1. 请求方式
    
        常见的请求方式有POST,GET,DELETE，但是一般使用的方式为`post`,`get`
    
        post:常用于从客户端上传信息
    
        get:常用于请求服务器某个路径下的某个资源，或者进行相关信息的查询。
    
        delete、put、options一般不会进行使用
    
    1. 请求内容
    
        一般GET方式用Params进行上传数据，并且上传的数据会直接在连接上展示。例如：`Key:key Value：myset`。则在请求链接上会有如下展示：`http://localhost:8080/scard?key=myset`，在服务端用gin框架解析时，可以使用`key := c.Query("key")`进行查询获取。
    
        当服务器需要进行token鉴权时，可以使用Authorization，常见的有Bearer Token，JWT Bearer,以及API Key,一般常用API Key，它可以自定义`Key`和`Value`，并且可以选择是将其加入进Header或者是Query Params。
    
        如果使用的是body进行传输数据，那么可以进行如下处理：
    
        1. **Form 表单数据**： 如果在Postman中以"form-data"形式发送请求，在Gin中可以使用 `c.PostForm` 方法来获取表单数据。
    
        2. **JSON 数据**： 如果在Postman中以"raw"格式发送JSON数据，在Gin中可以使用 `c.ShouldBindJSON` 方法来将JSON数据绑定到结构体中。Gin会自动解析JSON数据并填充到指定的结构体字段中。
    
        3. **Query 参数**： 如果在Postman中以"params"形式发送请求，在Gin中可以使用 `c.Query` 方法来获取查询参数。
    
        4. **Multipart/文件上传**： 如果在Postman中选择"form-data"并上传文件，在Gin中可以使用 `c.FormFile` 方法来处理文件上传。
    
        5. **X-www-form-urlencodea数据:**如果在Postman中选择"X-www-form-urlencodea"形式发送数据，gin可以使用
    
        ```Go
        // 获取单个参数
        value := c.PostForm("key")
        ```
    
        ```Go
        // 获取多个参数
        values, _ := c.PostFormArray("key")
        ```
    
        获取参数。
    
    1. postman的高级用法
    
        postman可以选择请求右侧的code,从而直接生成了请求代码。可以选择Documentation从而编辑API文档。
    
        postman提交文件，只能提交其工作目录下的内容，发送请求服务器才能读取的到

- 使用gin框架收到前端的数据该如何取进行提取？

    当在 Gin 框架中处理 HTTP 请求时，我们可以使用多种方式获取请求参数、头信息以及其他与请求相关的信息。

    1. **Query 参数：** 在 URL 中通过 `?key1=value1&key2=value2` 的形式传递参数，可以使用 `c.Query("key")` 方法获取单个参数的值，或者使用 `c.QueryMap()` 获取所有参数的键值对。

        ```Go
         name := c.Query("name")
         queryParams := c.QueryMap()
        ```

    1. **Form 表单参数：** 当客户端通过 POST 方法提交表单时，可以使用 `c.PostForm("key")` 方法获取单个参数的值，或者使用 `c.PostFormMap()` 获取所有表单参数的键值对。

        ```Go
         username := c.PostForm("username")
         formParams := c.PostFormMap()
        ```

    1. **JSON 参数：** 当客户端通过 POST 方法提交 JSON 数据时，可以使用 `c.ShouldBindJSON(&variable)` 方法将 JSON 数据绑定到相应的 Go 结构体变量上。

        ```Go
         var user User
         if err := c.ShouldBindJSON(&user); err != nil {
             // 处理错误
         }
        ```

    1. **路由参数：** 路由参数是指在路由路径中的占位符，例如 `/user/:id` 中的 `:id` 就是一个路由参数。可以通过 `c.Param("paramName")` 方法获取路由参数的值。

        ```Go
         userID := c.Param("id")
        ```

    1. **获取请求体数据：** 可以使用 `c.Request.Body` 来获取请求体的原始数据，然后进行解析或处理。

        ```Go
         body, err := ioutil.ReadAll(c.Request.Body)
        ```

    1. **Query 参数的默认值：** 使用 `c.DefaultQuery("key", "default")` 方法为没有传递的 Query 参数设置默认值。

        ```Go
         name := c.DefaultQuery("name", "Guest")
        ```

    1. **绑定并验证参数：** Gin 提供了 `ShouldBind` 或 `ShouldBindJSON` 方法将请求参数绑定到结构体，并支持参数的验证。

        ```Go
         var form LoginForm
         if err := c.ShouldBind(&form); err != nil {
             // 处理错误
         }
        ```

    1. **获取请求头信息：** 使用 `c.GetHeader("HeaderName")` 方法来获取请求头的值。

        ```Go
         contentType := c.GetHeader("Content-Type")
        ```

    1. **获取请求的 HTTP 方法：** 使用 `c.Request.Method` 可以获取当前请求的 HTTP 方法。

        ```Go
         httpMethod := c.Request.Method
        ```

    1. **获取请求路径和完整 URL：** 使用 `c.Request.URL.Path` 获取请求的路径，使用 `c.Request.URL.String()` 获取完整的请求 URL。

        ```Go
         path := c.Request.URL.Path
         fullURL := c.Request.URL.String()
        ```

    1. **获取 URL 参数（Path 参数）：** 使用 `c.Param("paramName")` 来获取路径参数的值。

        ```Go
         userID := c.Param("id")
        ```

    1. **获取多个相同名称的参数：** 使用 `c.QueryArray("key")` 或 `c.QueryMap("key")` 方法来获取多个相同名称的参数。

        ```Go
         roles := c.QueryArray("role")
         roleMap := c.QueryMap("role")
        ```

    1. **获取请求时间和 IP 地址：** 使用 `c.Request.Time` 获取请求的时间，而 `c.ClientIP()` 方法用于获取客户端的 IP 地址。

        ```Go
         requestTime := c.Request.Time
         clientIP := c.ClientIP()
        ```

    参考代码如下：

    ```Go
     package main
     
     import (
         "github.com/gin-gonic/gin"
         "io/ioutil"
         "net/http"
     )
     
     type User struct {
         Username string `json:"username"`
         Password string `json:"password"`
     }
     
     type LoginForm struct {
         User     string `form:"user" binding:"required"`
         Password string `form:"password" binding:"required"`
     }
     
     func main() {
         router := gin.Default()
     
         // 1. Query 参数
         router.GET("/query", func(c *gin.Context) {
             name := c.Query("name")
             age := c.Query("age")
             queryParams := c.QueryMap()
     
             c.JSON(200, gin.H{
                 "name":   name,
                 "age":    age,
                 "params": queryParams,
             })
         })
     
         // 2. Form 表单参数
         router.POST("/form", func(c *gin.Context) {
             username := c.PostForm("username")
             password := c.PostForm("password")
             formParams := c.PostFormMap()
     
             c.JSON(200, gin.H{
                 "username": username,
                 "password": password,
                 "params":   formParams,
             })
         })
     
         // 3. JSON 参数
         router.POST("/json", func(c *gin.Context) {
             var user User
     
             if err := c.ShouldBindJSON(&user); err != nil {
                 c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
                 return
             }
     
             c.JSON(200, gin.H{
                 "username": user.Username,
                 "password": user.Password,
             })
         })
     
         // 4. 路由参数
         router.GET("/user/:id", func(c *gin.Context) {
             userID := c.Param("id")
     
             c.JSON(200, gin.H{
                 "userID": userID,
             })
         })
     
         // 5. 获取请求体数据
         router.POST("/raw", func(c *gin.Context) {
             body, err := ioutil.ReadAll(c.Request.Body)
     
             if err != nil {
                 c.JSON(500, gin.H{"error": err.Error()})
                 return
             }
     
             c.JSON(200, gin.H{
                 "rawBody": string(body),
             })
         })
     
         // 6. Query 参数的默认值
         router.GET("/default", func(c *gin.Context) {
             name := c.DefaultQuery("name", "Guest")
     
             c.JSON(200, gin.H{
                 "name": name,
             })
         })
     
         // 7. 绑定并验证参数
         router.POST("/login", func(c *gin.Context) {
             var form LoginForm
     
             if err := c.ShouldBind(&form); err != nil {
                 c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
                 return
             }
     
             c.JSON(http.StatusOK, gin.H{
                 "user": form.User,
             })
         })
     
         // 8. 获取请求头信息
         router.GET("/header", func(c *gin.Context) {
             contentType := c.GetHeader("Content-Type")
     
             c.JSON(200, gin.H{
                 "Content-Type": contentType,
             })
         })
     
         // 9. 获取请求的 HTTP 方法
         router.Any("/method", func(c *gin.Context) {
             httpMethod := c.Request.Method
     
             c.JSON(200, gin.H{
                 "method": httpMethod,
             })
         })
     
         // 10. 获取请求路径和完整 URL
         router.GET("/path-url", func(c *gin.Context) {
             path := c.Request.URL.Path
             fullURL := c.Request.URL.String()
     
             c.JSON(200, gin.H{
                 "path":     path,
                 "full_url": fullURL,
             })
         })
     
         // 11. 获取 URL 参数（Path 参数）
         router.GET("/user/:id", func(c *gin.Context) {
             userID := c.Param("id")
     
             c.JSON(200, gin.H{
                 "userID": userID,
             })
         })
     
         // 12. 获取多个相同名称的参数
         router.GET("/multi-params", func(c *gin.Context) {
             roles := c.QueryArray("role")
             roleMap := c.QueryMap("role")
    
             c.JSON(200, gin.H{
                 "roles":    roles,
                 "role_map": roleMap,
             })
         })
     
         // 13. 获取请求时间和 IP 地址
         router.GET("/request-info", func(c *gin.Context) {
             requestTime := c.Request.Time
             clientIP := c.ClientIP()
     
             c.JSON(200, gin.H{
                 "request_time": requestTime,
                 "client_ip":     clientIP,
             })
         })
     
         // 启动服务
         router.Run(":8080")
     }
    ```

### 2.使用跨域的原因是什么？跨域是怎么样实现的？跨域使用的注意点有哪些？

    出于浏览器的同源策略限制。同源策略（Sameoriginpolicy）是一种约定，它是浏览器最核心也最基本的安全功能，如果缺少了同源策略，则浏览器的正常功能可能都会受到影响。可以说Web是构建在同源策略基础之上的，浏览器只是针对同源策略的一种实现。同源策略会阻止一个域的javascript脚本和另外一个域的内容进行交互。所谓同源（即指在同一个域）就是两个页面具有相同的协议（protocol），主机（host）和端口号（port）。
    
    跨域的实现，通过特定的规则，放行特定的内容，从而实现
    
    ```Go
    func Cors() gin.HandlerFunc {
    	return func(c *gin.Context) {
    		method := c.Request.Method               // 请求方法
    		origin := c.Request.Header.Get("Origin") // 请求头部
    		var headerKeys []string                  // 声明请求头keys
    		for k := range c.Request.Header {
    			headerKeys = append(headerKeys, k)
    		}
    		headerStr := strings.Join(headerKeys, ", ")
    		if headerStr != "" {
    			headerStr = fmt.Sprintf("access-control-allow-origin, access-control-allow-headers, %s", headerStr)
    		} else {
    			headerStr = "access-control-allow-origin, access-control-allow-headers"
    		}
    		if origin != "" {
    			c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
    			c.Header("Access-Control-Allow-Origin", "*")                                       // 这是允许访问所有域
    			c.Header("Access-Control-Allow-Methods", "POST, GET, OPTIONS, PUT, DELETE,UPDATE") // 服务器支持的所有跨域请求的方法,为了避免浏览次请求的多次'预检'请求
    			//  header的类型
    			c.Header("Access-Control-Allow-Headers", "Authorization, Content-Length, X-CSRF-Token, Token,session,X_Requested_With,Accept, Origin, Host, Connection, Accept-Encoding, Accept-Language,DNT, X-CustomHeader, Keep-Alive, User-Agent, X-Requested-With, If-Modified-Since, Cache-Control, Content-Type, Pragma")
    			// 允许跨域设置                                                                                                      可以返回其他子段
    			c.Header("Access-Control-Expose-Headers", "Content-Length, Access-Control-Allow-Origin, Access-Control-Allow-Headers,Cache-Control,Content-Language,Content-Type,Expires,Last-Modified,Pragma,FooBar") // 跨域关键设置 让浏览器可以解析
    			c.Header("Access-Control-Max-Age", "172800")                                                                                                                                                           // 缓存请求信息 单位为秒
    			c.Header("Access-Control-Allow-Credentials", "false")                                                                                                                                                  //  跨域请求是否需要带cookie信息 默认设置为true
    			c.Set("content-type", "application/json")                                                                                                                                                              // 设置返回格式是json
    		}
    		// 放行所有OPTIONS方法
    		if method == "OPTIONS" {
    			c.JSON(http.StatusOK, "Options Request!")
    		}
    		// 处理请求
    		c.Next() //  处理请求
    	}
    }
    
    ```
    
    跨域的注意点有哪些?
    
    1. **安全性：** 跨域请求涉及到安全性的考虑，确保只有信任的域名可以访问你的资源。通过设置适当的 CORS 头，限制允许访问的域名，避免开放过大的权限。
    
    2. **敏感信息：** 不要在跨域请求中暴露敏感信息。浏览器会执行同源策略以保护用户数据，但跨域请求可能导致一些敏感信息泄漏的风险。确保在跨域请求中不暴露不应该被公开的数据。
    
    3. **预检请求（Preflight Request）：** 对于一些复杂的跨域请求，浏览器会先发送一个 OPTIONS 请求（预检请求），以确认服务器是否支持实际请求所需的方法和头信息。服务器需要正确处理这个预检请求。
    
    4. **Cookie：** 跨域请求默认是不携带 Cookie 的，如果需要在跨域请求中发送 Cookie，需要确保服务器允许并在响应中设置 `Access-Control-Allow-Credentials` 头为 `true`，同时在客户端请求中设置 `withCredentials` 为 `true`。
    
    5. **方法限制：** 默认情况下，跨域请求只允许简单请求方法（GET、POST、HEAD），如果使用其他方法（如PUT、DELETE），会触发预检请求。服务器需要正确处理预检请求并在响应中指定允许的方法。
    
    6. **XSS（Cross-Site Scripting）攻击：** 跨域请求可能导致 XSS 攻击的风险。确保在处理跨域请求时，对输入进行适当的验证和转义，以防止恶意脚本的注入。
    
    7. **WebSocket 安全性：** 如果使用 WebSocket 进行跨域通信，同样需要考虑安全性问题。确保 WebSocket 连接通过加密（wss://）传输，以防止信息被窃听。
    
    8. **跨域资源共享策略的配置：** 在服务器端配置 CORS 头时，要仔细考虑允许的域名、方法和头信息。过于宽松的配置可能导致安全风险。



### 3.MySQL如何进行预编译防止SQL注入？

    - 使用预编译语句和参数化查询的方式来防止 SQL 注入攻击。预编译语句可以确保用户输入的数据不会被直接拼接到 SQL 查询语句中，从而降低 SQL 注入的风险。

### 4.gin框架的运行过程是什么？相较于go语言中的http包，有什么样的优点？

1. **创建 Gin 实例：** 应用程序首先创建一个 Gin 实例，作为整个应用的核心。

    ```Go
    goCopy code
    package main
    
    import "github.com/gin-gonic/gin"
    
    func main() {
        // 创建 Gin 实例
        r := gin.Default()
    
        // 定义路由和处理函数
    
        // 启动服务
        r.Run(":8080")
    }
    ```

1. **定义路由和处理函数：** Gin 使用路由来映射 HTTP 请求到相应的处理函数。开发者可以使用不同的 HTTP 方法（GET、POST、PUT、DELETE 等）和 URL 路径来定义路由规则，并指定相应的处理函数。

    ```Go
    goCopy code
    // 示例路由定义
    r.GET("/hello", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "Hello, World!"})
    })
    ```

1. **启动服务：** 通过调用 `Run` 方法，Gin 框架会启动一个 HTTP 服务器，开始监听指定的端口，等待客户端请求。

    ```Go
    goCopy code
    r.Run(":8080")
    ```

1. **处理请求和响应：** 当有 HTTP 请求到达时，Gin 框架会根据请求的路径和方法找到匹配的路由规则，并调用相应的处理函数。处理函数可以读取请求参数、执行业务逻辑，最终返回响应给客户端。

Gin 框架相较于 Go 语言标准库中的 `http` 包有一些优点：

1. **更快的性能：** Gin 被设计为高性能框架，相对于 `http` 包，它在处理请求时更加快速。Gin 使用了 Radix 树来实现路由匹配，提高了路由查找的效率。

2. **中间件支持：** Gin 提供了中间件的机制，可以方便地扩展应用的功能，例如日志记录、认证、跨域处理等。中间件可以按照顺序组合使用，使得代码结构更加清晰。

3. **更友好的路由定义：** Gin 的路由定义方式更为简洁和灵活，支持参数匹配、通配符等功能，使得路由配置更加方便。

4. **JSON 集成：** Gin 提供了方便的 JSON 处理方法，可以轻松地将结构体转换为 JSON 格式的响应，或者将请求中的 JSON 数据解析成结构体。

5. **框架级别的上下文管理：** Gin 提供了上下文（Context）对象，其中包含了 HTTP 请求和响应的相关信息，方便在处理函数中获取请求参数、设置响应头等操作。



