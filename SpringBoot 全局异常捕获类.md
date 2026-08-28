# SpringBoot 全局异常捕获类

核心注解：`@RestControllerAdvice` + `@ExceptionHandler`

> 作用：统一捕获 Controller 抛出的异常，不用每个接口写 try‑catch；**只拦截 Controller 层抛出的异常，Service 内部自己吞掉的异常捕获不到**。

## 1、完整全局异常类代码

```
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import lombok.Data;

// 全局控制器增强，作用于所有@RestController
@RestControllerAdvice
public class GlobalExceptionHandler {

    // ----------------------1.捕获自定义业务异常----------------------
    @ExceptionHandler(BusinessException.class)
    public Result<?> handleBusinessException(BusinessException e){
        return Result.fail(e.getCode(),e.getMessage());
    }

    // ----------------------2.捕获系统通用异常(兜底)----------------------
    @ExceptionHandler(Exception.class)
    public Result<?> handleException(Exception e){
        // 生产环境可以打印日志 e.printStackTrace();
        return Result.fail(500,"服务器内部异常");
    }

    // 还可以细分：NullPointerException、MethodArgumentNotValidException 参数校验异常等
}

// 统一返回体
@Data
class Result<T>{
    private Integer code;
    private String msg;
    private T data;

    public static <T> Result<T> fail(Integer code,String msg){
        Result<T> r=new Result<>();
        r.setCode(code);
        r.setMsg(msg);
        return r;
    }
}

// 自定义业务异常
class BusinessException extends RuntimeException{
    private Integer code;
    public BusinessException(Integer code,String msg){
        super(msg);
        this.code=code;
    }
    public Integer getCode(){return code;}
}
```

## 2、它是怎么生效的（底层原理）

1. `@RestControllerAdvice`
   - 组合注解，包含 `@ControllerAdvice` + `@ResponseBody`
   - Spring 启动时，这个类会被扫描，变成 IOC 容器中的 Bean。
   - **作用范围：拦截所有 @RestController / @Controller 的控制器**。
2. `@ExceptionHandler(异常类.class)`
   - 这是回调规则：当 Controller 方法抛出**匹配的异常对象**，Spring MVC 就去找这个被`@ExceptionHandler`标记的方法执行。
   - 匹配规则：**优先精确匹配，再向上找父类**。比如抛出`BusinessException`，优先走`BusinessException`处理器；如果没有，就找父类`RuntimeException`，最后兜底`Exception`。

> ⚠️关键点：

1. **只捕获 Controller 抛出往外抛的异常！**

- 如果你的 Service 写了 try‑catch，异常被吃掉，没有向上抛给 Controller，**全局异常处理器不会触发**。

```
// 不会进全局异常！异常被吞了
public void test(){
    try{
        int i=1/0;
    }catch (Exception e){
        e.printStackTrace();
    }
}
// ✅会进全局异常：异常向上抛出
public void test(){
    int i=1/0;
}
```

1. **不会拦截过滤器 (Filter) 抛出的异常**Filter 在 Controller 之前执行，`@RestControllerAdvice`管不到 Filter。Filter 异常要另外处理。
2. 自定义业务异常使用示例

```
@RestController
public class TestController{
    @GetMapping("/test")
    public void test(){
        throw new BusinessException(400,"参数错误");
    }
}
```

接口抛出，就会被`GlobalExceptionHandler`捕获，返回统一 json。

## 面试高频问题

1. `@ControllerAdvice`和`@RestControllerAdvice`区别

- `@ControllerAdvice`：返回视图；
- `@RestControllerAdvice`：自带`@ResponseBody`，直接返回 JSON，前后端分离项目用这个。

1. 全局异常能不能捕获 AOP 里抛出的异常？✅**可以**，AOP 是在 controller 执行前后，AOP 抛出异常会交给 ControllerAdvice 处理。
2. 优先级：精确异常 > 父类异常。写多个`@ExceptionHandler`，抛出的异常会优先匹配最精确的那个处理器。

## 简单流程总结

1. Spring 扫描`@RestControllerAdvice`类，注册为 Bean。
2. Controller 执行，抛出异常。
3. SpringMVC 寻找`@ExceptionHandler`，匹配异常类型。
4. 执行对应方法，把返回值转为 JSON 返回前端。