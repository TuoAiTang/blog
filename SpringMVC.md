SpringMVC

# 基本概念

- DispatcherServlet
	前端控制器

- Handler
	DispatcherServlet调用Controller的过渡对象

- HandlerAdapter 
	适配器

- HandlerIntercepter
	Handler 拦截器

- HandlerMapping
	帮助 DispatcherServlet 找到正确的 Controller
	用 HandlerIntercepter 包装 Controller

- HandlerExecutionChain
	preHandle -> Controller method	->	postHandle	-> afterCompletion

- ModelAndView
	Model 或 Map 都会被DispatcherServlet转化为ModelAndView

- ViewResolver
	Help DispatcherServlet ro resolve the right View to render page.

- View
	Responsible for page rendering.