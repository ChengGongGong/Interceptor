# 注解形式Interceptor 
# 1.自定义注解
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Target({ElementType.TYPE, ElementType.METHOD})
@Order(Ordered.HIGHEST_PRECEDENCE)
public@interface TestFilter {
    boolean value() default true;
}

# 2.自定义类实现HandlerInterceptor接口，并重新其中的方法

@Slf4j
public class TestInterceptor implements HandlerInterceptor {
    private final static String ENCODING = "UTF-8";

    private final static String CONTENT_TYPE = "application/json; charset=utf-8";

    @Autowired
    private XxxService xxxService;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        log.info("Interceptor started{} ");
        boolean result=true;
        HandlerMethod handlerMethod= (HandlerMethod) handler;
        Method method=handlerMethod.getMethod();
        TestFilter verifyFilter=method.getAnnotation(TestFilter.class);
        if(verifyFilter==null){
            verifyFilter=handlerMethod.getBeanType().getAnnotation(TestFilter.class);
        }
        if(verifyFilter==null ||!verifyFilter.value()){
            return true;
        }
        String timestamp = request.getHeader("timeStamp");
        //请求时间戳参数为空
        if (StringUtils.isEmpty(timestamp)) {
            JsonVO r=new JsonVO(ReturnCode.timeEmptyCode, ReturnCode.timeEmptyMsg);
            resultStr(r,response);
            return false;
        }
        return result;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception{
        JsonVO result=xxxService.getXxx();
        log.info("Interceptor:{} ",result);
    }

    /**
     * 返回相关信息
     *  @param r
     * @param response
     */
    private void resultStr(JsonVO r, HttpServletResponse response) {
        response.setCharacterEncoding(ENCODING);
        response.setContentType(CONTENT_TYPE);
        String responseStr = new Gson().toJson(r);
        try (PrintWriter out=response.getWriter()){
            out.append(responseStr);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

# 3.注册该bean到spring中去

@Configuration
public class FilterConfig extends WebMvcConfigurationSupport {
    
    //此处手动注入interceptor，防止该拦截器中注入serviceBean出现空指针异常
    @Bean
    public TestInterceptor testInterceptor() {
      return new TestInterceptor();
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
       // registry.addInterceptor(new TestInterceptor())
        registry.addInterceptor(testInterceptor())
                .addPathPatterns("/**")
                .excludePathPatterns("/test");

    }

}

# springboot集成filter 
# 1.使用注解WebFilter直接集成

@Slf4j
@Component
@WebFilter(filterName = "CommonFilter", urlPatterns = "/*")
public class CommonFilter extends GenericFilter {
    @Autowired
    private PropertyManager propertyManager;

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;
        String path = request.getRequestURI();
        String[] whitePaths = propertyManager.getSignWhitePaths();
        boolean allowedPath = false;
        for (String whitePath : whitePaths) {
            if (StringUtils.equals(whitePath, path)) {
                allowedPath = true;
                break;
            }
        }
        if (!allowedPath) {
           //具体业务过滤逻辑 
        }
        filterChain.doFilter(servletRequest, servletResponse);
    }
}

# 2.使用bean注入的方式

@Component
public class CommonFilterConfig  {
    @Autowired
    private CommonFilter commonFilter;
    //如果有多个filter，注入多个bean
    @Bean
    public FilterRegistrationBean<commonFilterr> commonFilterRegister{
        FilterRegistrationBean<CommonFilter> registration = new FilterRegistrationBean<>();
        registration.setFilter(commonFilter);
        registration.addUrlPatterns("/*");
        registration.setName("commonFilter");
        //值越小，Filter越靠前
        registration.setOrder(1);
        return registration;
    }
}

# 总结Interceptor(拦截器)与filter(过滤器)的区别
1.实现原理不同，过滤器基于函数回调的，实现doFilter()方法,拦截器是基于java反射机制(动态代理)实现的。
    
2.使用范围不同，过滤器实现的是javax.servelet.Filter接口，需要依赖于Tomcat等web容器，只能在web程序中使用，
  而拦截器是一个spring组件，由spring容器管理，并不依赖于web容器，可以单独使用，也可以用于Application、Swing等程序中；
    
3.触发时机不同，过滤器是在请求进入容器后，进入servelet之前进行预处理，请求结束是在servlet处理完成之后，
  而拦截器是在请求进入servlet之后，进入Controller之前进行预处理的，请求结束是在controller处理完成之后；
    
4.拦截的请求范围不同，过滤器几乎对所有进入容器的请求起作用，而拦截器只会对controller中的请求或访问static目录下的资源请求起作用。
    
5.注入bean的情况不同，可以在过滤器中直接注入service服务并生效，而拦截器需要先将interceptor手动注入，再在拦截器中注入service服务,
  因为拦截器加载的时间点是在springcontext之前，而bean是由spring进行管理的。
    
6.控制执行顺序不同，过滤器用@Order注解控制执行顺序，通过@Order控制过滤器的级别，值越小级别越高越先执行；
  而拦截器默认的执行顺序，就是它的注册顺序，也可以通过Order手动设置控制，值越小越先执行，先声明的拦截器 preHandle() 方法先执行，而postHandle()方法反而会后执行。
  

