# 双拦截器无感刷新token

分析：单个拦截器，只拦截需要用户登录才能访问的页面，假如用户登录之后，访问不需要拦截的页面，比如说主页，拦截器并不会执行，token也不会刷新，即使用户正在使用token因为长时间不刷新回过期。

解决：可以在引入一个拦截器，拦截所有的请求，该拦截器的主要目的就是刷新token,

第一个拦截器: 主要是刷新token,并将对应的用户信息保存在ThreadLocal,这样第二个拦截器直接可以从ThreadLocal获取值。

```java
/**
 * 刷新token拦截器，拦截所有
 *
 * */
public class RefleshTokenInterceptor implements HandlerInterceptor {


    private StringRedisTemplate stringRedisTemplate;

    public RefleshTokenInterceptor(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        //  1. 获取token
        String token = request.getHeader("authorization");
        if (StrUtil.isBlank(token)) {
            //为空 放行
//            response.setStatus(401);
            return  true;
        }

        // 2.基于token获取redis中的用户
        Map<Object, Object> objectMap = stringRedisTemplate.opsForHash().entries(RedisConstants.LOGIN_USER_KEY+token);
        //判断用户是否存在
        if (objectMap.isEmpty()){
            //不存在 放行
//            response.setStatus(401);
            return true;
        }
        //将map转对象
        User user = BeanUtil.fillBeanWithMap(objectMap, new User(), false);

        //保存再threadLocal
        UserHolder.saveUser(user);

        //刷新token有效期
        stringRedisTemplate.expire(RedisConstants.LOGIN_USER_KEY+token,RedisConstants.LOGIN_USER_TTL, TimeUnit.MINUTES);

        return true;

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
       //移除信息，避免内存泄露
        UserHolder.removeUser();
    }
}

```

第二个拦截器

```java
/**
 * 拦截器，拦截需要认证的业务
 */
public class LoginInterceptor implements HandlerInterceptor {


    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        // todo: 单拦截器  session登录方式
//        //1. 获取session
//        HttpSession session = request.getSession();
//        // 2. 获取session中的用户
//        Object user = session.getAttribute("user");
//        if (user == null) {
//            //没有用户信息
//            response.setStatus(401);
//            return false;
//        }
//        //3. 保存到ThreadLocal中
//        UserHolder.saveUser((User) user);
//
//        return true;


        // todo: 单拦截器 token登录方式
//        // TODO 1. 获取token
//        String token = request.getHeader("authorization");
//        if (StrUtil.isBlank(token)) {
//            //为空
//            response.setStatus(401);
//            return  false;
//        }
//
//        // TODO 2.基于token获取redis中的用户
//        Map<Object, Object> objectMap = stringRedisTemplate.opsForHash().entries(RedisConstants.LOGIN_USER_KEY+token);
//        //判断用户是否存在
//        if (objectMap.isEmpty()){
//            //不存在
//            response.setStatus(401);
//            return false;
//        }
//        //将map转对象
//        User user = BeanUtil.fillBeanWithMap(objectMap, new User(), false);
//
//        //保存再threadLocal
//        UserHolder.saveUser(user);
//
//        //刷新token有效期
//        stringRedisTemplate.expire(RedisConstants.LOGIN_USER_KEY+token,RedisConstants.LOGIN_USER_TTL, TimeUnit.MINUTES);

        // todo 修改拦截器为双拦截器后的代码，直接根据全拦截器看ThreadLocal的值判断是否拦截
        User user = UserHolder.getUser();
        if (user == null) {
            //没有用户信息 需要拦截
            response.setStatus(401);
            return  false;
        }
        return true;

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
       //移除信息，避免内存泄露
        UserHolder.removeUser();
    }
}
```

拦截器注册: 需要注意的是拦截器执行的先后，可以设置order属性默认是0，数字越小优先级越高。

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Resource
    private StringRedisTemplate stringRedisTemplate;


    //登录拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor())
                .excludePathPatterns(
                        "/user/code",
                        "/user/login",
                        "/shop-type/**",
                        "/api/shop/**",
                        "/blog/hot"
                ).order(1);

        //刷新token拦截器
        registry.addInterceptor(new RefleshTokenInterceptor(stringRedisTemplate))
                .addPathPatterns("/**")
                .order(0);
    }
}
```

