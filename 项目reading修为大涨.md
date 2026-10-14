# 项目reading修为大涨

1 看一个项目最先看controller层，因为controller是最接近人的语言，相当于接口文档一比一复刻

![image-20261008211130161](C:\Users\ThinkPad\AppData\Roaming\Typora\typora-user-images\image-20261008211130161.png)

①是类名

②是类里面的内置类

③是bean

相当于声明一个类，然后这个类是通过spring注入实现的

③是①的第一个字母变小写（spring的规则）

**元注解**

| Kotlin                             | Java                    | 能贴在哪  |
| :--------------------------------- | :---------------------- | :-------- |
| `AnnotationTarget.FUNCTION`        | `ElementType.METHOD`    | 方法/函数 |
| `AnnotationTarget.CLASS`           | `ElementType.TYPE`      | 类        |
| `AnnotationTarget.FIELD`           | `ElementType.FIELD`     | 字段      |
| `AnnotationTarget.VALUE_PARAMETER` | `ElementType.PARAMETER` | 参数      |

### 建造者模式

分步定义对象，通过链式调用定义最终对象

### 被Kotlin的伴生对象取代

类：模板

实例：根据模板建立的具体东西

## 伴生对象的作用

* #### 类级别共享

  **伴生对象在这里的作用，是给 `AuditLogAspect` 提供"类级别共享的无状态工具资源"（Logger、SpEL 解析器、参数名发现器），让它们全局唯一、随类加载初始化、按 SLF4J 惯例组织，避免随实例重复创建，同时保持封装性。**

  ```java
  package studio.lingrui.security.audit
  
  import com.fasterxml.jackson.databind.ObjectMapper
  import jakarta.servlet.http.HttpServletRequest
  import org.aspectj.lang.ProceedingJoinPoint
  import org.slf4j.LoggerFactory
  import org.aspectj.lang.annotation.Around
  import org.aspectj.lang.annotation.Aspect
  import org.aspectj.lang.reflect.MethodSignature
  import org.slf4j.MDC
  import org.springframework.core.DefaultParameterNameDiscoverer
  import org.springframework.expression.ExpressionParser
  import org.springframework.expression.spel.standard.SpelExpressionParser
  import org.springframework.expression.spel.support.StandardEvaluationContext
  import org.springframework.stereotype.Component
  import org.springframework.web.context.request.RequestContextHolder
  import org.springframework.web.context.request.ServletRequestAttributes
  import org.springframework.web.multipart.MultipartFile
  import studio.lingrui.core.exception.AppException
  import studio.lingrui.core.log.AuditLog
  import studio.lingrui.core.log.ParamSanitizer
  
  /**
   * 审计日志切面
   *
   * 拦截所有标注了 [@AuditLog] 的方法，记录：
   * - 请求参数（脱敏后）
   * - 耗时
   * - 成功/失败状态
   * - 错误码和错误信息（失败时）
   *
   * objectId 支持 SpEL 表达式：
   * - "#id" — 取方法参数中的 id
   * - "#req.userId" — 取参数对象的嵌套属性
   * - 纯字符串 — 直接使用
   */
  @Aspect
  @Component
  class AuditLogAspect(
      private val writer: SecurityAuditLogWriter,
      private val objectMapper: ObjectMapper,
  ) {
      companion object {
          private val log = LoggerFactory.getLogger(AuditLogAspect::class.java)
          private val parser: ExpressionParser = SpelExpressionParser()
          private val nameDiscoverer = DefaultParameterNameDiscoverer()
      }
  
      // IDEA会对 HttpServletRequest 的注入报错，因为它是由Tomcat在运行时注入的，IDEA的Spring插件没法找到对应的 Bean
      // 你可以完全忽略IDEA报错，也可以显式地说明其为@Lazy
      // 或者像我在这里一样，用Spring自带的 RequestContextHolder
      private fun getRequest(): HttpServletRequest {
          return (RequestContextHolder.getRequestAttributes() as ServletRequestAttributes).request
      }
  
      @Around("@annotation(auditLog)")
      fun audit(proceedingJoinPoint: ProceedingJoinPoint, auditLog: AuditLog): Any? {
          val start = System.currentTimeMillis()
          val requestId = MDC.get("requestId")
          val request = getRequest()
  
          val method = request.method
          val path = request.requestURI
          val controller = "${proceedingJoinPoint.target.javaClass.simpleName}.${proceedingJoinPoint.signature.name}"
          val params = extractAndSanitizeParams(proceedingJoinPoint)
          val objectId = resolveObjectId(auditLog.objectId, proceedingJoinPoint)
  
          try {
              val result = proceedingJoinPoint.proceed()
              val duration = (System.currentTimeMillis() - start).toInt()
  
              writer.write(
                  requestId = requestId,
                  operation = auditLog.operation,
                  objectId = objectId,
                  detail = auditLog.detail.ifEmpty { null },
                  method = method,
                  path = path,
                  params = params,
                  controller = controller,
                  status = 0,
                  errorCode = null,
                  errorMessage = null,
                  duration = duration,
              )
  
              return result
          } catch (e: Exception) {
              val duration = (System.currentTimeMillis() - start).toInt()
  
              writer.write(
                  requestId = requestId,
                  operation = auditLog.operation,
                  objectId = objectId,
                  detail = auditLog.detail.ifEmpty { null },
                  method = method,
                  path = path,
                  params = params,
                  controller = controller,
                  status = 1,
                  errorCode = (e as? AppException)?.errorCode?.code,
                  errorMessage = e.message,
                  duration = duration,
              )
  
              throw e
          }
      }
  
      /**
       * 解析 objectId，支持 SpEL 表达式
       *
       * "#id" → 取方法参数 id 的值
       * "#req.userId" → 取参数对象 req 的 userId 属性
       * "abc" → 直接返回 "abc"
       */
      private fun resolveObjectId(expression: String, pjp: ProceedingJoinPoint): String? {
          if (expression.isEmpty()) return null
  
          if (!expression.startsWith("#")) {
              return expression
          }
  
          return try {
              val signature = pjp.signature as MethodSignature
              val paramNames = nameDiscoverer.getParameterNames(signature.method) ?: return expression
              val paramValues = pjp.args
  
              val context = StandardEvaluationContext()
              paramNames.zip(paramValues).forEach { (name, value) ->
                  context.setVariable(name, value)
              }
  
              parser.parseExpression(expression).getValue(context, String::class.java)
          } catch (e: Exception) {
              log.warn("SpEL 解析失败: expression={}, error={}", expression, e.message)
              expression
          }
      }
  
      private fun extractAndSanitizeParams(pjp: ProceedingJoinPoint): String? {
          return try {
              val signature = pjp.signature as MethodSignature
              val names = signature.parameterNames
              val values = pjp.args
  
              val params = names.zip(values).toMap()
                  .filter { (_, value) -> value !is MultipartFile }  // MultipartFile 过滤放在 Adapter 层
              val sanitized = ParamSanitizer.sanitize(params)
              objectMapper.writeValueAsString(sanitized)
          } catch (_: Exception) {
              null
          }
      }
  }
  
  ```

  

* #### 工厂模式

  **工厂方法必须是类级别的 → Kotlin 里类级别的方法只能放伴生对象 → 所以工厂方法放伴生对象里。**

  工厂方法就是**一个函数，它的职责是"造出某个对象并返回"**

  要造一个对象就不能先要一个对象，所以工厂方法必须是类级别的

  

```java
ackage studio.lingrui.core.response

import com.fasterxml.jackson.annotation.JsonInclude
import studio.lingrui.core.exception.ErrorCodeProvider

/**
 * 统一返回的数据类，这次给时间戳日期这些东西都省略了，后续再加，
 * 同时把Builder模式改为了直接使用Kotlin的伴生对象，太伟大了kotlin
 *
 * 成功: {"code": 0, "data": {...}}  （成功的时候，message千篇一律的成功，省略掉就好了）
 * 失败: {"code": 3001, "message": "题目不存在"} （异常的时候，就没有数据返回了）
 */
@JsonInclude(JsonInclude.Include.NON_NULL)
data class ApiResponse<T>(
    val code: Int,
    val message: String? = null,
    val data: T? = null
) {
    companion object {
        fun success(): ApiResponse<Unit> =
            ApiResponse(code = 0)

        fun <T> success(data: T): ApiResponse<T> =
            ApiResponse(code = 0, data = data)

        /**
         * 从 ErrorCodeProvider 构建失败响应
         * GlobalExceptionHandler 中:
         *   ApiResponse.fail(e.errorCode, e.detail)
         */
        fun fail(errorCode: ErrorCodeProvider, detail: String? = null): ApiResponse<Unit> =
            ApiResponse(code = errorCode.code, message = detail ?: errorCode.message)

        /**
         * 直接传 code 和 message (用于非 ErrorCodeProvider 场景)，
         * 后面模块建议先枚举好错误码（参照exception包中的CommonErrorCode）
         */
        fun fail(code: Int, message: String): ApiResponse<Unit> =
            ApiResponse(code = code, message = message)
    }
}

```



### 整洁架构/六边形架构

```
studio.lingrui.announcement
├── adapter        ← 对外接口（Controller、RepositoryImpl）
├── application    ← 应用层：用例编排  ★
├── config         ← 配置
├── domain         ← 领域层：核心业务规则
├── redis          ← 缓存基础设施
└── websocket      ← 实时通信
```



