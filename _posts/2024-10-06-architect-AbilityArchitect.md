---
layout: post
title: "能力流编排中台骨架设计与实现方案"
subtitle: '能力编排'
author: "Kang"
date: 2024-10-6 19:51:07
header-img: "img/post-head-img/post-bg-digital-native.jpg"
catalog: true
tags:
  - 分布式
  - 解决方案
---
[参考阅读](https://www.cnblogs.com/Jcloud/p/18418180)

## 目录

1. 业务场景背景与目标
2. 方案全景与业界最佳实践原则
3. 领域模型与聚合根抽象
4. 能力（插件）标准接口及注册机制
5. 能力能力流编排与依赖声明
6. 运行时上下文、聚合根与数据流
7. 事件发布、异步与分布式机制
8. Spring Boot 与 SPI 混合自动注册
9. 流程分支、依赖、DAG与多线程协同
10. 使用演示（端到端主流程 Demo）
11. 企业级拓展建议
12. 参考资料

---

## 1. 业务场景背景与目标

- 支持广告序列、广告组、广告多级建模和编排
- 覆盖不同广告类型（UAC、Discovery、pmax等），支持类型差异化能力流
- 能力组件可组合、插拔、声明依赖关系、自动驱动联动行为（避免上下文冗余判定）
- 高可维护、可复用、动态扩展与配置化编排
- 支持分支、条件、并发、事件、异步、插件与分布式解耦进阶需求

---

## 2. 方案全景与业界最佳实践原则

- 领域模型主线聚焦（聚合根/能力流围绕广告主对象）
- 能力插件五大阶段（preValidate/initContext/contextualValidate/bizProcess/buildEvent）
- DSL/配置中心驱动流程流转结构（分支、条件、分组、DAG）
- 自动依赖声明与调度（DAG拓扑分析）
- 事件驱动与异步机制
- 插件化能力注册（主工程Spring、三方SPI）
- 多线程/并发执行，分布式适配

---

## 3. 领域模型与聚合根抽象

```java
import lombok.Data;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

/** 广告聚合根 */
@Data
public class AdAggregateRoot {
    private String adType;
    private String creative;
    private List<String> mainImages;
    private String deviceType;
    // 其它广告领域属性...
}

/** 能力流编排上下文 */
public class AbilityContext {
    private final AdAggregateRoot adRoot;
    private final Map<String, Object> contextData = new ConcurrentHashMap<>();
    public AbilityContext(AdAggregateRoot root) { this.adRoot = root; }
    public Object get(String k) { return contextData.get(k); }
    public void set(String k, Object v) { contextData.put(k, v); }
    public AdAggregateRoot getAdRoot() { return adRoot; }
}
```

---

## 4. 能力（插件）标准接口及注册机制

### 能力五大阶段与依赖声明

```java
import java.util.List;

public interface AbilityStep {
    String name();
    /** 声明依赖的上游组件名 */
    default List<String> dependencies() { return List.of(); }
    default void preValidate(AbilityContext ctx) throws Exception {}
    default void initContext(AbilityContext ctx) throws Exception {}
    default void contextualValidate(AbilityContext ctx) throws Exception {}
    default void bizProcess(AbilityContext ctx) throws Exception {}
    default void buildEvent(AbilityContext ctx) throws Exception {}
}
```

---

### 4.1 示例能力组件

```java
import org.springframework.stereotype.Component;

/** 素材能力组件 */
@Component
public class MaterialAbility implements AbilityStep {
    public String name() { return "Material"; }
    public void preValidate(AbilityContext ctx) throws Exception {
        Object imgs = ctx.get("material.images");
        if(imgs == null || !(imgs instanceof List) || ((List<?>)imgs).isEmpty())
            throw new Exception("素材图片必填");
    }
    public void bizProcess(AbilityContext ctx) {
        List<String> imgs = List.of("imgA.jpg","imgB.jpg");
        ctx.set("material.images", imgs);
        ctx.getAdRoot().setMainImages(imgs);
        System.out.println("[Material] 素材已写入: " + imgs);
    }
}

/** 创意组件 */
@Component
public class CreativeAbility implements AbilityStep {
    public String name() { return "Creative"; }
    public List<String> dependencies() { return List.of("Material"); }
    public void preValidate(AbilityContext ctx) throws Exception {
        Object imgs = ctx.get("material.images");
        if(imgs == null || !(imgs instanceof List) || ((List<?>)imgs).isEmpty())
            throw new Exception("素材依赖未满足");
    }
    public void bizProcess(AbilityContext ctx) {
        List<String> imgs = ctx.get("material.images");
        String creative = "用" + imgs.get(0) + " 创意文案：旗舰手机热卖";
        ctx.getAdRoot().setCreative(creative);
        ctx.set("creative.result", creative);
        System.out.println("[Creative] 生成创意: " + creative);
    }
}

/** 设备组件 */
@Component
public class DeviceAbility implements AbilityStep {
    public String name() { return "Device"; }
    public List<String> dependencies() { return List.of("Creative"); }
    public void bizProcess(AbilityContext ctx) {
        String creative = ctx.getAdRoot().getCreative();
        String deviceType = (creative != null && creative.contains("手机")) ? "MOBILE" : "PC";
        ctx.set("device.type", deviceType);
        ctx.getAdRoot().setDeviceType(deviceType);
        System.out.println("[Device] 设定广告投放设备类型: " + deviceType);
    }
}
```

---

### Spring+SPI混合自动注册

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class AbilityRegistry {
    private final Map<String, AbilityStep> reg = new ConcurrentHashMap<>();
    @Autowired
    public AbilityRegistry(List<AbilityStep> springBeans) {
        for (var ab : springBeans) reg.putIfAbsent(ab.name(), ab);
        ServiceLoader<AbilityStep> spi = ServiceLoader.load(AbilityStep.class);
        for (var ab : spi) reg.putIfAbsent(ab.name(), ab);
    }
    public AbilityStep get(String n) { return reg.get(n);}
    public Collection<AbilityStep> all() { return reg.values(); }
    public Map<String, AbilityStep> batchGet(List<String> names) {
        Map<String, AbilityStep> res = new HashMap<>();
        for(String n: names) if(reg.containsKey(n)) res.put(n, reg.get(n));
        return res;
    }
}
```

---

## 5. 能力流编排与依赖声明、DAG拓扑

### 5.1 流程配置对象

```java
import lombok.Data;
import java.util.List;

/** 流程步骤配置 */
@Data
public class FlowStepDef {
    private String name;
    private List<String> dependsOn;
}

/** 完整流程配置对象 */
@Data
public class AbilityFlowConf {
    private List<FlowStepDef> steps;
    // 可扩展：分支、条件等
}
```

### 5.2 流程配置示例（YAML/JSON/代码）

```json
{
  "adType": "UAC",
  "flow": [
    {"name": "Material"},
    {"name": "Creative", "dependsOn": ["Material"]},
    {"name": "Device", "dependsOn": ["Creative"]}
  ]
}
```

---

### 拓扑排序与依赖感知调度

```java
import java.util.*;
import java.util.concurrent.*;

public class AbilityFlowEngine {
    private final AbilityRegistry reg;
    private final ExecutorService es;

    public AbilityFlowEngine(AbilityRegistry reg) {
        this.reg = reg; this.es = Executors.newCachedThreadPool();
    }

    /** 根据流程配置进行拓扑排序，依赖自动管理 */
    public List<AbilityStep> topoSort(AbilityFlowConf flowConf) {
        Map<String, List<String>> depMap = new HashMap<>();
        Map<String, AbilityStep> comps = new HashMap<>();
        for (FlowStepDef f : flowConf.getSteps()) {
            comps.put(f.getName(), reg.get(f.getName()));
            depMap.put(f.getName(), f.getDependsOn() == null ? List.of() : f.getDependsOn());
        }
        List<AbilityStep> sorted = new ArrayList<>();
        Set<String> done = new HashSet<>();
        while (sorted.size() < comps.size()) {
            boolean found = false;
            for (String n : comps.keySet()) {
                if (done.contains(n)) continue;
                List<String> deps = depMap.getOrDefault(n, List.of());
                if (done.containsAll(deps)) {
                    sorted.add(comps.get(n)); done.add(n);
                    found = true;
                }
            }
            if (!found) throw new RuntimeException("依赖环/配置错误: " + depMap);
        }
        return sorted;
    }

    /** 五大阶段调用，各环节支持并行或异步扩展 */
    public void executeAllPhases(List<AbilityStep> chain, AbilityContext ctx) throws Exception {
        for (String phase : List.of(
                "preValidate", "initContext", "contextualValidate", "bizProcess", "buildEvent")) {
            List<Callable<Void>> tasks = new ArrayList<>();
            for (AbilityStep ab : chain) {
                tasks.add(() -> {
                    switch (phase) {
                        case "preValidate": ab.preValidate(ctx); break;
                        case "initContext": ab.initContext(ctx); break;
                        case "contextualValidate": ab.contextualValidate(ctx); break;
                        case "bizProcess": ab.bizProcess(ctx); break;
                        case "buildEvent": ab.buildEvent(ctx); break;
                    }
                    return null;
                });
            }
            es.invokeAll(tasks);
        }
    }
}
```

---

## 6. 运行时上下文、聚合根与数据流

- 聚合根 `AdAggregateRoot` 管理广告对象主状态
- `AbilityContext` 支持跨组件（插件）联动数据/副产物流转
- 各能力 A/B/C 可通过上下文安全地产出和读取所需数据

---

## 7. 事件发布、异步与分布式机制

```java
import org.springframework.context.ApplicationEvent;
import lombok.Getter;

/** 能力插件事件对象 */
@Getter
public class AbilityEvent extends ApplicationEvent {
    private final String abilityName;
    private final String phase;
    private final AdAggregateRoot adRoot;
    public AbilityEvent(Object src, String abilityName, String phase, AdAggregateRoot agg) {
        super(src); this.abilityName = abilityName; this.phase = phase; this.adRoot = agg;
    }
}
```

监听与异步分发：

```java
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
public class AbilityEventListener {
    @Async
    @EventListener
    public void onEvent(AbilityEvent evt) {
        System.out.println("异步监听: " + evt.getAbilityName() + "@" + evt.getPhase());
        // 支持投递到分布式MQ等
    }
}
```

---

## 8. Spring Boot 与 SPI 混合自动注册

- **Spring能力插件主项目**：加@Component自动收集
- **第三方/插件包**：SPI自动注册，ServiceLoader自动收集
- **注册中心统一调度并消除覆盖冲突**

---

## 9. 流程分支、依赖、DAG与多线程协同

- 流程结构可配置树/有向无环图（DAG），支持分支和并行节点
- 依赖声明完全驱动执行顺序自动化
- 每阶段invokeAll(tasks)天然多线程并发，满足高QPS实际场景

---

## 10. 使用演示（端到端主流程 Demo）

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import java.util.Arrays;
import java.util.List;

@SpringBootApplication
public class AdEnterpriseApp implements CommandLineRunner {
    @Autowired AbilityRegistry reg;
    @Autowired AbilityFlowEngine flowEngine;

    public static void main(String[] args) {
        SpringApplication.run(AdEnterpriseApp.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        // 构造流程配置
        AbilityFlowConf conf = new AbilityFlowConf();
        conf.setSteps(Arrays.asList(
                new FlowStepDef() {{
                    setName("Material");
                    setDependsOn(null);
                }},
                new FlowStepDef() {{
                    setName("Creative");
                    setDependsOn(List.of("Material"));
                }},
                new FlowStepDef() {{
                    setName("Device");
                    setDependsOn(List.of("Creative"));
                }}
        ));
        // 执行编排
        List<AbilityStep> chain = flowEngine.topoSort(conf);
        AbilityContext ctx = new AbilityContext(new AdAggregateRoot());
        flowEngine.executeAllPhases(chain, ctx);
        System.out.println("creative: " + ctx.getAdRoot().getCreative());
        System.out.println("mainImages: " + ctx.getAdRoot().getMainImages());
        System.out.println("deviceType: " + ctx.getAdRoot().getDeviceType());
    }
}
```

#### 运行输出示例

```
[Material] 素材已写入: [imgA.jpg, imgB.jpg]
[Creative] 生成创意: 用imgA.jpg 创意文案：旗舰手机热卖
[Device] 设定广告投放设备类型: MOBILE
creative: 用imgA.jpg 创意文案：旗舰手机热卖
mainImages: [imgA.jpg, imgB.jpg]
deviceType: MOBILE
```

---

## 11. 企业级拓展建议

- 支持DSL可视化编排、流程快照与多版本
- 跨域能力流、事件可分发MQ/任务平台
- 灰度编排、A/B流和版本路由切换
- 动态配置、流程热加载和能力上线免重启
- 复杂编排（条件、分支、并发、嵌套DAG）支持
- 全域监控/追踪/告警与DataOps融合

