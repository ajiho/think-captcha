# think-captcha

基于thinkphp6的一个图形验证码拓展包


# 效果图

![效果图](https://img-blog.csdnimg.cn/6a52de7b48ac401c855ff6d5482bc3dc.png)
![效果图](https://img-blog.csdnimg.cn/afadf851243149068aa5f11798976397.png)
![效果图](https://img-blog.csdnimg.cn/041f9d0ed7c2456ca5342baa7f98c1e4.png)


# 优点

- 简单灵活、相对于thinkphp官方的验证码没有过渡包装
- 只提供验证码的生成、没有限定验证码的存储方式,完全自己掌控
- 相对于thinkphp官方的验证码在前后端分离开发时不需要改源码进行适配
- 不用在项目中引入验证码所需的字体文件

# 安装

```
composer require ajiho/think-captcha
```

# 配置

安装完毕后会自动生成配置文件 /config/captcha.php

```php
<?php
return [
    //随机因子(已排除容易混淆的1和iI、0和oO)
    'charset' => 'abcdefghkmnprstuvwxyzABCDEFGHKMNPRSTUVWXYZ23456789',
    //验证码长度
    'codelen' => '4',
    //宽度
    'width' => '150',
    //高度
    'height' => '50',
    //字体大小
    'fontsize' => '20',
];
```


# 使用


## mvc开发方式

### 生成验证码

首先在控制器方法中生成验证码，用于给页面进行显示

```php
//new验证码对象
$captcha = new \ajiho\Captcha();
//这里大家可以根据情况选择存储方式,这里用session做演示
session('captcha', $captcha->getCode());
//输出图片
return $captcha->outImg();
```

### 检验验证码

假设前端表单传递的验证码的name值为`captcha`

```php

//接受参数
$captcha = input('captcha','');

//判断表单传递过来的验证码和上面存到session中的验证码是否一致
if ($captcha !== session('captcha')) {
   //验证码不正确
}
//如果你不想区分大小写，可以统一都转换为小写再对比
//if (strtolower($captcha) !== strtolower(session('captcha'))) {

//验证通过
//session('captcha', null); //验证通过后根据需要 是否要删除存在session的验证码

```


## 前后端分离开发使用

### 流程说明

在使用之前先说一下大致流程(用登录案例说明):

1. 客户端请求验证码接口
2. 返回结果(展示图形验证码的url地址(指向后端的验证码展示方法)、`uniqid`)
3. 把url地址赋值给img的src属性,同时把`uniqid`存储起来待会儿用于表单提交
4. 当img图片显示的时候就会触发后端验证码展示方法，该接口方法会把`uniqid`作为键，把验证码作为值缓存起来
5. 客户端登录信息输入完毕后携带参数(账号、密码..、验证码、`uniqid`)去请求登录接口
6. 在后台登录处理方法中通过`uniqid`去缓存中查找对应的验证码和表单提交的验证码进行对比是否正确，至此流程结束

### 代码流程

#### 1.你需要定义两个接口
如果你没有使用强制路由，可忽略，这里定义出来是为了方便说明

```php
//该路由是用于验证码显示
Route::get('captcha/:uniqid', 'Captcha/show');

//验证码接口
Route::get('captcha', 'Captcha/create');

//登录处理
Route::post('login', 'Login/login');
```



#### 2.验证码接口方法
```php
//验证码唯一标识
$uniqid = uniqid((string)mt_rand(100000, 999999));
//说明:这里生成的链接其实就是指向验证码显示方法
$src = (string)\think\facade\Route::buildUrl('/captcha/' . $uniqid)->domain(true);

$data = [
    'src' => $src,
    'uniqid' => $uniqid,
];

//这里你们可以根据自己封装的api接口返回方法返回数据
return Api::ok($data);
```




#### 3.验证码显示方法

```php
public function show($uniqid)
{
 	$captcha = new \ajiho\Captcha();
    //存缓存,时间短一点，减少压力
    cache('captcha_' . $uniqid, $captcha->getCode(), 120);
    //输出图像
    return $captcha->outImg();
}
```


#### 4.登录处理方法里面验证

```php
public function login(Request $request)
{
	$params = $request->all();

    //表单验证
    try {
        validate([
            'username|用户名' => 'require|alphaDash',
            'password|密码' => 'require|alphaDash',
            'uniqid|验证码标识' => 'require',
            'code|验证码' => 'require',
        ])->check($params);

        //从缓存中根据uniqid取验证码
        $code = cache('captcha_' . $params['uniqid']);
        if (!$code) {//没取到说明验证码过期
            return Api::fail('验证码过期,请刷新后重试');
        }
        //对比缓存中的验证码和表单传递过来的验证码是否相等
        if ($params['code'] !== $code) {
            return Api::fail('验证码错误');
        }

    } catch (ValidateException $e) {
        return Api::fail($e->getError());
    }
	
	//验证通过...向下继续执行你的登录逻辑
}
```

# 反馈

如果有任何问题或者建议，可以直接到仓库留言。

