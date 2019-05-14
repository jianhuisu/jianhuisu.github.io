---
title : "Yii的RESTful API的应用解析"
categories : 
 - php 
tags :
	- PHP
	- Yii2.0
---

Tips:本文为笔记,内容多引用他人。侵删

Yii 提供了一整套用来简化实现 RESTful 风格的 Web Service 服务的 API

`RESTful API`是关于资源的操作,什么是资源呢,资源就是存储在服务器上各式各样的数据。
这些数据资源有的存储在数据库中,如最基本`user`表,还有一些不适合在数据库中存储。

在如何代表一个资源没有固定的限定，在 Yii 中通常使用 yii\base\Model 或它的子类（如 yii\db\ActiveRecord） 代表资源，是为以下原因：

- yii\base\Model 实现了 yii\base\Arrayable 接口， 它允许你通过 RESTful API 自定义你想要公开的资源数据。
- yii\base\Model 支持 输入验证, 在你的 RESTful API 需要支持数据输入时非常有用。
- yii\db\ActiveRecord 提供了强大的数据库访问和操作方面的支持， 如资源数据需要存到数据库它提供了完美的支持。

## RESTful API在Yii中的实现

例如,如果我们想要提供一套操作`admin`表的RESTful API,我们需要如下操作

Step.1 创建控制器

	<?php

	namespace frontend\controllers;

	use yii\rest\ActiveController;

	class AdminController extends ActiveController
	{
		public $modelClass = 'frontend\models\Admin';
	}
	
		
Step.2 创建控制器生成 `frontend\models\Admin` AR类

	<?php
	namespace frontend\models;

	use yii\db\ActiveRecord;

	class Admin extends ActiveRecord
	{
		public static function tableName()
		{
			return 'admin';
		}

	}		
	
`frontend\models\Admin`是基于`admin`表的ActiveRecord类。	

Step.3 配置文件中启用Url美化(nginx中先开启rewrite)

	'urlManager' => [
            'enablePrettyUrl'     => true,
            'enableStrictParsing' => false, // 这个推荐设为 false 严格解析代表如果 规则没有预定义则拒绝访问
            'showScriptName'      => false,
            'rules' => [
                ['class' => 'yii\rest\UrlRule', 'controller' => 'admin'],
            ],
        ],	

`urlManager`组件会在路由解析之前调用,按照预定义的规则进行匹配
	
	public function parseRequest($request)
    {
        if ($this->enablePrettyUrl) {
           
			...
            return [$pathInfo, []];
        }

        Yii::debug('Pretty URL not enabled. Using default URL parsing logic.', __METHOD__);
        $route = $request->getQueryParam($this->routeParam, '');
        if (is_array($route)) {
            $route = '';
        }

        return [(string) $route, []];
    }

解析优先级为 预定义规则 > 默认规则
	
只需要这三步,可以通过以下调用方式实现对`admin`表的CRUD操作。
	
	GET      /users:       逐页列出所有用户
	HEAD     /users:       显示用户列表的概要信息
	POST     /users:       创建一个新用户
	GET      /users/123:   返回用户 123 的详细信息
	HEAD     /users/123:   显示用户 123 的概述信息
	PATCH    /users/123:   更新用户123
	PUT      /users/123:   更新用户123
	DELETE   /users/123:   删除用户123
	OPTIONS  /users:       显示关于末端 /users 支持的动词
	OPTIONS  /users/123:   显示有关末端 /users/123 支持的动词

现在如果要在后台管理系统中开发一个管理员模块,分分钟搞定。	

但是在实际工作中,单表操纵可谓少之又少,很多时候资源都是多表数据汇总之后经过聚合之后的结果,
并且,查询操作的调用频率、查询维度的多样性都要远大于其它三种操作，依靠AR提供的通用操作接口显然远远不够
这时我们就需要自定义RESTful资源操作接口。

	<?php
	
	namespace frontend\controllers;

	use yii\filters\RateLimiter;
	use yii\rest\Controller;
	use yii\web\Response;

	class FooController extends Controller
	{
		public $modelClass = 'frontend\models\FooForm';

		public function behaviors() {

			$behaviors = parent::behaviors();
			$behaviors['rateLimiter'] = [
				'class'                  => RateLimiter::className(),
				'enableRateLimitHeaders' => true,
			];

			// 如果你将 RESTful APIs 作为应用开发，可以设置应用配置中 Response 组件的相应格式
			// 如果将 RESTful APIs 作为模块开发，可以在模块的 init() 方法中设置响应格式
			$behaviors['contentNegotiator']['formats']['text/html'] = Response::FORMAT_JSON;
			
			return $behaviors;

		}

		public function actionView($id)
		{
			echo __FUNCTION__;
		}

		public function actionSearch()
		{
			echo __FUNCTION__;
		}

		...

	}
	
urlManager中添加规则
	
	'urlManager' => [
            'enablePrettyUrl'     => true,
            'enableStrictParsing' => false,
            'showScriptName'      => false,
            'rules' => [
                [
                    'class' => 'yii\rest\UrlRule',
                    'controller' => 'foo',
                    'extraPatterns' => [
                        'GET  search' => 'search',
                    ]
                ],
                [
                    'class' => 'yii\rest\UrlRule',
                    'controller' => 'admin'
                ],
            ],
        ],

然后就可以通过get方式访问http://hostname/foos/search 接口		
	
## 限流 Rate Limiting

在开发高并发系统时有三把利器用来保护系统：缓存、降级和限流

- 缓存 缓存的目的是提升系统访问速度和增大系统处理容量
- 降级 降级是当服务出现问题或者影响到核心流程时，需要暂时屏蔽掉，待高峰或者问题解决后再打开
- 限流 限流的目的是通过对并发访问/请求进行限速，或者对一个时间窗口内的请求进行限速来保护系统，一旦达到限制速率则可以拒绝服务、排队、降级等处理

Yii默认为RESTful API提供了限流功能,该限流针对于用户.User表中使用两列来记录容差和时间戳信息
	
	allowance              剩余的允许请求数(原英文意思为限量,限额)
	allowance_updated_at   最后一次接口访问时间戳	      
	
User中需要先实现RateLimitInterface中的三个方法(建议换到NoSQL存储,这里只为说明限流的实现算法,并未切换),实现对这两个字段的读写

	public function getRateLimit($request, $action)
	{
		return [$this->rateLimit, 1]; // 预定义 每个User最大每秒请求数
	}

	public function loadAllowance($request, $action)
	{
		return [$this->allowance, $this->allowance_updated_at];
	}
	
	public function saveAllowance($request, $action, $allowance, $timestamp)
	{
		$this->allowance = $allowance;
		$this->allowance_updated_at = $timestamp;
		$this->save();
	}

然后RateLimite过滤器中会按照如下规则进行调用,
	
	/**
     * Checks whether the rate limit exceeds.
     * @param RateLimitInterface $user the current user
     * @param Request $request
     * @param Response $response
     * @param \yii\base\Action $action the action to be executed
     * @throws TooManyRequestsHttpException if rate limit exceeds
     */
    public function checkRateLimit($user, $request, $response, $action)
    {
        list($limit, $window) = $user->getRateLimit($request, $action);
        list($allowance, $timestamp) = $user->loadAllowance($request, $action);

        $current = time();
		
		// 这一句是核心 使用令牌桶算法 $limit 为令牌桶的最大容量 
        $allowance += (int) (($current - $timestamp) * $limit / $window);   
        if ($allowance > $limit) {
            $allowance = $limit;
        }

        if ($allowance < 1) {
            $user->saveAllowance($request, $action, 0, $current);
            $this->addRateLimitHeaders($response, $limit, 0, $window);
            throw new TooManyRequestsHttpException($this->errorMessage);
        }

        $user->saveAllowance($request, $action, $allowance - 1, $current);
        $this->addRateLimitHeaders($response, $limit, $allowance - 1, (int) (($limit - $allowance + 1) * $window / $limit));
    }
	
常用的限流算法有

- 计数器
- 滑动窗口
- 令牌桶
- 漏桶

Google开源项目Guava中的RateLimiter使用的就是令牌桶控制算法。

漏桶算法

把请求比作是水，水来了都先放进桶里，并以限定的速度出水，当水来得过猛而出水不够快时就会导致水直接溢出,请求被丢弃,即拒绝服务。

漏斗有一个进水口 和 一个出水口，出水口以一定速率出水，并且有一个最大出水速率：

在漏斗中没有水的时候，

- 如果进水速率小于等于最大出水速率，那么，出水速率等于进水速率，此时，不会积水
- 如果进水速率大于最大出水速率，那么，漏斗以最大速率出水，此时，多余的水会积在漏斗中

在漏斗中有水的时候,出水口以最大速率出水

- 如果漏斗未满，且有进水的话，那么这些水会积在漏斗中
- 如果漏斗已满，且有进水的话，那么这些水会溢出到漏斗之外

令牌桶算法

对于很多应用场景来说，除了要求能够限制数据的平均传输速率外，还要求允许某种程度的突发传输。这时候漏桶算法可能就不合适了，令牌桶算法更为适合。
令牌桶算法的原理是系统以恒定的速率产生令牌，然后把令牌放到令牌桶中，令牌桶有一个容量，当令牌桶满了的时候，再向其中放令牌，那么多余的令牌会被丢弃；
当想要处理一个请求的时候，需要从令牌桶中取出一个令牌，如果此时令牌桶中没有令牌，那么则拒绝该请求。

令牌桶算法VS漏桶算法

漏桶

漏桶的出水速度是恒定的，那么意味着如果瞬时大流量的话，将有大部分请求被丢弃掉（也就是所谓的溢出）。

令牌桶

生成令牌的速度是恒定的，而请求去拿令牌是没有速度限制的。这意味，面对瞬时大流量，该算法可以在短时间内请求拿到大量令牌，而且拿令牌的过程并不是消耗很大的事情。

Tips:不论是对于令牌桶拿不到令牌被拒绝，还是漏桶的水满了溢出，都是为了保证大部分流量的正常使用，而牺牲掉了少部分流量，这是合理的，
如果因为极少部分流量需要保证的话，那么就可能导致系统达到极限而挂掉，得不偿失。分布式环境下可用使redis限流。

## 认证(authorization)与授权(authorize) 

### 认证

RESTful APIs 通常是无状态的， 也就意味着不应使用 sessions 或 cookies，
 因此每个请求应附带某种授权凭证，因为用户授权状态可能没通过 sessions 或 cookies 维护，
常用的做法是每个请求都发送一个秘密的 access token 来认证用户， 由于 access token 可以唯一识别和认证用户,所以api尽量使用https协议。
如何在RESTful api中设置认证方式呢？


Step.1 RESTful的主调控制器中设置认证方式
	
	DefaultController extends \yii\rest\Controller
	...
	
	 public function behaviors()
    {
        $behaviors = parent::behaviors();
        $behaviors['contentNegotiator']['formats']['text/html'] = Response::FORMAT_JSON;
        $behaviors['authenticator'] = [
            'class' => HttpBasicAuth::className(),
        ];

        return $behaviors;
    }
	...
		

Step.2 在User components将`findIdentityByAccessToken()`方法实现

	public static function findIdentityByAccessToken($token, $type = null)
    {
        // 可以将 token 与 user 的映射关系在redis中维护
        return static::findOne(['access_token' => $token]);
    }


Yii常见的认证方式有三种

-  HttpBasicAuth   header头中传递username/password  
-  HttpBearerAuth  通过Bear token在header中搭载token
-  QueryParamAuth  通过url参数"access-token=ASKJFDjjdd_dD3DDFDFDWDDDD"传递token
		
### 版本化
	
Yii建议api采用增量发布的方式,这样可以向后兼容

	api/
    common/
        controllers/
            UserController.php
            PostController.php
        models/
            User.php
            Post.php
    modules/
        v1/
            controllers/
                UserController.php
                PostController.php
            models/
                User.php
                Post.php
            Module.php
        v2/
            controllers/
                UserController.php
                PostController.php
            models/
                User.php
                Post.php
            Module.php
	

Yii的模块时可以重复嵌套的,所以可以对一个模块下的接口增量发布

	car/
        v1/
            controllers/
                UserController.php
                PostController.php
            models/
                User.php
                Post.php
            Module.php
        v2/
            controllers/
                UserController.php
                PostController.php
            models/
                User.php
                Post.php
            Module.php
	
	
## 参考资料

Yiichian https://www.yiichina.com/doc/guide/2.0/rest-quick-start

限流算法  搜云库技术团队 https://www.jianshu.com/p/dd1071e08469