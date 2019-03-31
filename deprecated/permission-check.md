### 基于AOP和自定义注解实现的权限验证
> 在项目建立初期时使用的权限验证实现, 如今有更好的验证方式, 将原先的逻辑实现贴在此处, 写过的代码即便旧了也不怎么想直接删掉, 还是有些参考意义的

#### 自定义注解
``` java
package com.welearn.annotation;

import java.lang.annotation.*;

@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface PermissionCheck {
    // 权限名称，使用此注解标注的方法参与权限验证
    // 如未填写则按照服务名-业务类名-方法名默认命名
    // 所有该注解标注的业务层方法会被拦截进行权限验证
    // 同时服务 会定时获取更新 获取角色权限关联数据
    // 对于用户的请求，其用户角色关联数据会缓存 以提供高效率的权限验证
    String name() default "";
    String description() default "";
    // userId 是否先尝试从方法参数中进行读取
    boolean isCheckMethodUserArg() default true;
    // 验证失败提示的错误信息
    String message() default "user in group auth check failed to do it";
}
```


#### AOP逻辑验证
``` java
package com.welearn.interceptor;

import com.welearn.annotation.PermissionCheck;
import com.welearn.dictionary.api.ServiceConst;
import com.welearn.dictionary.authorization.RequestTypeConst;
import com.welearn.entity.po.common.User;
import com.welearn.entity.vo.request.common.CheckPermissionCode;
import com.welearn.entity.vo.response.authorization.AuthResult;
import com.welearn.error.exception.PermissionCheckFailedException;
import com.welearn.error.exception.ProgramCheckFailedException;
import com.welearn.error.exception.UserAuthCheckFailedException;
import com.welearn.feign.common.PermissionFeignClient;
import com.welearn.util.GlobalContextUtil;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.util.Objects;
import java.util.Set;

import static com.welearn.dictionary.authorization.RequestTypeConst.REQUEST_FROM_ROUTE;
import static com.welearn.dictionary.authorization.RequestTypeConst.REQUEST_FROM_SYSTEM_INSIDE;
import static com.welearn.dictionary.common.RequestHeaderConst.REQUEST_TYPE;

/**
 * Description : 权限检查处理AOP
 * Created by Setsuna Jin on 2018/1/31.
 *
 * 该类被废弃, 权限的验证逻辑将被移至 route 服务中处理
 * Created by Setsuna Jin on 2019/03/31.
 */
@Slf4j
@Aspect
@Component
public class PermissionCheckProcessAOP {

    public static final String PERMISSION_NAME_TEMPLATE = "%s/%s/%s";

    @Value("${debug}")
    private Boolean isDebug;

    @Value("${spring.application.name}")
    private String applicationName;

    @Autowired
    private PermissionFeignClient permissionFeignClient;

    @Around("execution(* com.welearn.service..*(..)) && @annotation(permissionCheck)")
    public Object Interceptor(ProceedingJoinPoint point, PermissionCheck permissionCheck) throws Throwable {
        ServiceConst service = ServiceConst.get(applicationName);

        String userId = null;

        boolean isCheckMethodUserArg = permissionCheck.isCheckMethodUserArg();

        // 是否从参数中获取组与用户信息
        MethodSignature methodSignature = (MethodSignature) point.getSignature();
        if (isCheckMethodUserArg){
            Class[] paramTypes = methodSignature.getParameterTypes();
            try {
                for (int i = 0; i < paramTypes.length; i++) {
                    if (Objects.isNull(userId) && paramTypes[i] == User.class)
                        userId = ((User) point.getArgs()[i]).getId();
                }
            } catch (Exception e){
                throw new PermissionCheckFailedException("analyse user from method args failed", e);
            }
        }
        // 生成权限名称
        String permissionCode = String.format(PERMISSION_NAME_TEMPLATE,
                service.getServiceName(), point.getTarget().getClass().getSimpleName().replace("ServiceImpl",""), methodSignature.getMethod().getName());

        // 如果参数中无用户信息 尝试从请求中获取
        AuthResult authResult = null;
        RequestTypeConst requestType = REQUEST_FROM_ROUTE;
        // 按照当前请求的用户信息获取 userId 读取请求中的 AuthResult 进行权限验证
        if (Objects.isNull(userId)){
            try {
                String requestTypeName = GlobalContextUtil.getRequest().getHeader(REQUEST_TYPE.getHeaderName());
                // 服务间通过FeignClient相互调用
                if (REQUEST_FROM_SYSTEM_INSIDE.name().equals(requestTypeName))
                    requestType = RequestTypeConst.REQUEST_FROM_SYSTEM_INSIDE;
                if (requestType.equals(REQUEST_FROM_ROUTE)){
                    authResult = GlobalContextUtil.getAuthResult();
                }
            }
            catch (ProgramCheckFailedException e){
                // 服务内部业务层Bean相互调用
                requestType = RequestTypeConst.REQUEST_FROM_SYSTEM_INSIDE;
            }
            catch (Exception e){
                e.printStackTrace();
                if (requestType.equals(REQUEST_FROM_ROUTE))
                    throw new UserAuthCheckFailedException("permission check failed", e);
            }
        }
        // 读取方法参数中的 userId 进行权限验证
        else {
            CheckPermissionCode checkPermissionCode = new CheckPermissionCode();
            checkPermissionCode.setUserId(userId);
            checkPermissionCode.setPermissionCode(permissionCode);
            Boolean isCheckPermissionFailed = permissionFeignClient.checkPermissionCode(checkPermissionCode).getData();
            if (isCheckPermissionFailed)
                throw new UserAuthCheckFailedException("user:{} have no permission:{}", userId, permissionCode);
        }

        // 执行权限的检查
        if (Objects.nonNull(authResult)){
            Set<String> userHavePermissionCodeList = authResult.getPermissionCodeList();
            if (!userHavePermissionCodeList.contains(permissionCode))
                throw new UserAuthCheckFailedException("user have no permission:{}", permissionCode);
        }
        // 执行拦截的方法
        return point.proceed();
    }
}
```

#### 权限的自动更新
``` java
package com.welearn.task;

import com.welearn.annotation.PermissionCheck;
import com.welearn.dictionary.api.ServiceConst;
import com.welearn.dictionary.common.PermissionTypeConst;
import com.welearn.entity.po.BasePersistant;
import com.welearn.entity.po.common.Permission;
import com.welearn.entity.po.common.ReRolePermission;
import com.welearn.entity.po.common.Role;
import com.welearn.entity.vo.request.common.BindPermissions;
import com.welearn.entity.vo.response.common.RoleRefPermissions;
import com.welearn.feign.common.PermissionFeignClient;
import com.welearn.feign.common.RoleFeignClient;
import com.welearn.interceptor.PermissionCheckProcessAOP;
import com.welearn.util.PackageScannerUtil;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.type.classreading.MetadataReader;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.lang.reflect.Method;
import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

/**
 * Description : 服务权限初始化及更新
 * Created by Setsuna Jin on 2018/4/14.
 */
@Slf4j
@Component
public class PermissionInitTask {

    @Autowired
    private PermissionFeignClient permissionFeignClient;

    @Autowired
    private RoleFeignClient roleFeignClient;

    @Value("${spring.application.name}")
    private String applicationName;

    private static final String SERVICE_IMPL_PACKAGE = "com.welearn.service.impl";

    @Scheduled(initialDelay = 1000*60, fixedRate = 1000*60*60*24)
    public void run() throws Exception {
        this.processUpdateServicePermissions();
        this.processAdminRoleBindAllPermissions();
    }

    /**
     * 检验当前代码架构里的权限信息 和 数据库 是否一致, 如有改变对其进行 添加 启用 或 禁用
     * @throws IOException
     * @throws ClassNotFoundException
     */
    private void processUpdateServicePermissions() throws IOException, ClassNotFoundException {
        log.info("START RUN PERMISSION INIT TASK");
        List<Permission> servicePermissions =
                permissionFeignClient.selectServicePermission(applicationName).getData();

        Map<String, Permission> servicePermissionIndex = new HashMap<>();
        for (Permission permission : servicePermissions) {
            servicePermissionIndex.put(permission.getCode(), permission);
        }
        Set<String> dbPermissionNames = servicePermissionIndex.keySet();
        Map<String, Permission> currentPermissions = this.getPackageCurrentPermissions();
        Set<String> currentPermissionNames = currentPermissions.keySet();

        System.out.println(servicePermissions.size());

        // 取得交集， 对未启用权限执行启用
        Set<String> intersections = new HashSet<>(dbPermissionNames);
        intersections.retainAll(currentPermissionNames);
        for (String permissionName : intersections){
            Permission permission = servicePermissionIndex.get(permissionName);
            if (!permission.getIsEnable())
                permissionFeignClient.enable(permission.getId());
        }
        log.info("ENABLED OLD PERMISSION COUNT:{}", intersections.size());
        // 取得数据库中权限和当前权限的差集, 对启用权限执行未启用
        Set<String> difference = new HashSet<>(dbPermissionNames);
        difference.removeAll(intersections);
        for (String permissionName : difference){
            Permission permission = servicePermissionIndex.get(permissionName);
            if (permission.getIsEnable())
                permissionFeignClient.disable(permission.getId());
        }
        log.info("DISABLE OLD PERMISSION COUNT:{}", difference.size());
        // 取得当前权限和数据库中权限的差集, 对其执行添加权限操作
        difference.clear();
        difference.addAll(currentPermissionNames);
        difference.removeAll(intersections);
        for (String permissionName : difference){
            permissionFeignClient.create(currentPermissions.get(permissionName));
        }
        log.info("CREATED NEW PERMISSION COUNT:{}", difference.size());
    }

    /**
     * 当权限信息发生变化时, 自动更新 系统管理员 角色 关联所有权限
     */
    private void processAdminRoleBindAllPermissions(){
        Role adminRole = roleFeignClient.selectByCode("admin").getData();
        if (!Objects.isNull(adminRole)){
            log.info("START UPDATE ADMIN ROLE BIND PERMISSION");
            RoleRefPermissions rps = permissionFeignClient.roleRefPermissions().getData();
            roleFeignClient.bindPermissions(new BindPermissions(
                    adminRole.getId(),
                    rps.getPermissions().stream().map(BasePersistant::getId).collect(Collectors.toList())
            ));
            log.info("ADMIN ROLE BIND PERMISSION COUNT:{}", rps.getPermissions().size());
        }
    }

    private Map<String, Permission> getPackageCurrentPermissions() throws IOException, ClassNotFoundException {
        ServiceConst service = ServiceConst.get(applicationName);
        Map<String, Permission> permissionNames = new HashMap<>();
        Set<MetadataReader> serviceImplClasses = PackageScannerUtil.doScan(SERVICE_IMPL_PACKAGE);
        for (MetadataReader serviceImplClass : serviceImplClasses) {
            Class clazz = Class.forName(serviceImplClass.getClassMetadata().getClassName());
            if ("BaseServiceImpl".equals(clazz.getSimpleName()))
                continue;
            Method[] serviceMethods = clazz.getMethods();
            for (Method serviceMethod : serviceMethods) {
                if (!serviceMethod.isAnnotationPresent(PermissionCheck.class))
                    continue;
                PermissionCheck check = serviceMethod.getAnnotation(PermissionCheck.class);
                String code = String.format(
                        PermissionCheckProcessAOP.PERMISSION_NAME_TEMPLATE,
                        service.getServiceName(), clazz.getSimpleName().replace("ServiceImpl",""), serviceMethod.getName());
                String name = StringUtils.isBlank(check.name()) ? code : check.name();
                String description = StringUtils.isBlank(check.description()) ? "" : check.description();

                Permission p = new Permission();
                p.setCode(code);
                p.setName(name);
                p.setService(applicationName);
                p.setType(PermissionTypeConst.SERVICE_PERMISSION.ordinal());
                p.setDescription(description);

                permissionNames.put(code, p);
            }
        }
        return permissionNames;
    }
}

```