# think-captcha

基于thinkphp6的一个图形验证码拓展包


# 效果图

![效果图](https://img-blog.csdnimg.cn/6a52de7b48ac401c855ff6d5482bc3dc.png)
![效果图](https://img-blog.csdnimg.cn/afadf851243149068aa5f11798976397.png)
![效果图](https://img-blog.csdnimg.cn/041f9d0ed7c2456ca5342baa7f98c1e4.png)


# 优点

- 简单灵活、相对于tp官方的验证码没有过渡包装
- 只提供验证码的生成、没有限定验证码的存储方式,前后端分离开发时不需要改源码进行适配
- 不用在项目中引入验证码所需的字体文件


# 安装

```
composer require ajiho/think-captcha
```

# 使用



## 生成验证码

首先在控制器方法中生成验证码，用于给页面进行显示

```php
//new验证码对象
$captcha = new \ajiho\Captcha();
//这里大家可以根据情况选择存储方式,这里用session做演示
session('captcha', $captcha->getCode());
//输出图片
return $captcha->outImg();
```

## 检验验证码

假设前端表单传递的验证码的name值为`captcha`

```php

//接受参数
$params = input();

//判断表单传递过来的验证码和上面存到session中的验证码是否一致
if ($params['captcha'] != session('captcha')) {
   //验证码不正确
}

//验证通过...

//验证通过后根据需要 是否要删除存在session的验证码
//session('captcha', null);

```


# 配置

/config/captcha.php

```php
<?php
return [
    //随机因子
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


# 前后端分离使用说明

前后端分离开发和该包本身没有什么太大的关系，用别的验证码也是同样的原理，只不过该包它比较简单、干净，易于使用


## 比如这里一个登录流程(原理)

1. 客户端请求验证码接口
2. 返回结果(页面用于展示图形验证码的url地址、`uniqid`)
3. 把url地址赋值给img的src属性,同时把`uniqid`存储起来
4. 当img图片显示的时候就已经触发验证码展示接口，该接口方法会把`uniqid`作为缓存的键，把验证码作为缓存值缓存起来
5. 请求登录接口(账号、密码..、验证码、`uniqid`)
6. 后台通过`uniqid`去缓存中查找对应的验证码和提交的验证码进行对比是否正确
7. 账号密码、验证码等一系列逻辑对比都通过后会把用户的id生成jwt返回给客户端登录成功

## 详细参考代码

这是对上面原理的具体实现，在使用的时候灵活变通

~~~
//验证码图形展示接口
Route::get('captcha/:uniqid', 'Validate/showCaptcha');

//验证码接口
Route::get('captcha', 'Validate/captcha');

public function showCaptcha()
{

    $params = input();

    try {
        validate([
            'uniqid|验证码标识' => 'require|alphaNum|length:32',
        ])->check($params);
    } catch (ValidateException $e) {
        return fail($e->getError());
    }

    $captcha = new Captcha();
    //存缓存,时间短一点，减少压力
    cache('captcha_' . $params['uniqid'], $captcha->getCode(), 60);
    return $captcha->outImg();

}


public function captcha()
{
    //验证码唯一标识
    $uniqid = md5(uniqid((string)mt_rand(100000, 999999)));
    //注意，这里其实就是生成验证码展示接口的url
    $src = (string)\think\facade\Route::buildUrl('/captcha/' . $uniqid)->domain(true);
    $data = [
        'src' => $src,
        'uniqid' => $uniqid,
    ];
    return ok($data);
}



//验证验证码是否正确部分
$params = input();

//表单验证
try {
    validate([
        'username|用户名' => 'require|alphaDash',
        'password|密码' => 'require|alphaDash',
        'uniqid|验证码标识' => 'require|alphaNum|length:32',
        'code|验证码' => 'require|checkCode',
    ])->check($params);

    //验证码验证
    $cache_code = cache('captcha_' . $params['uniqid']);
    if (!$cache_code) {
       //'验证码过期,请刷新后重试'
    }
    if ($params['code'] !== $cache_code) {
       //'验证码错误'
    }
    
    //对比账号密码...

} catch (ValidateException $e) {
   //...
}

//验证通过...
~~~


# 反馈

如果有任何问题或者建议，可以直接到仓库留言。

