# Phân tích khả năng nâng cấp Memcached Operator

## Tóm tắt Executive Summary

**✅ KHẢ THI** - Operator này có thể nâng cấp để quản lý cả memcached instances và proxy.

**Lý do:**
- Kiến trúc hiện tại đã sử dụng controller pattern chuẩn của Kubernetes
- Code đã có sẵn logic để tạo Deployment (cho mcrouter) - có thể tái sử dụng pattern này cho memcached
- Infrastructure (informers, workqueues, event handling) đã sẵn sàng để mở rộng

---

## 1. Phân tích kiến trúc hiện tại

### 1.1. Cấu trúc controller hiện tại (main.go:93-268)

Operator hiện có **4 controllers song song**:

```
┌─────────────────────────────────────────────────────────────┐
│                    MemcachedProxy CRD                        │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
        ┌──────────────────┴──────────────────┐
        │                                     │
        ▼                                     ▼
┌───────────────┐                    ┌────────────────┐
│ proxy         │                    │ proxyservice   │
│ controller    │                    │ controller     │
│               │                    │                │
│ Apply         │                    │ Create Service │
│ defaults      │                    │ for mcrouter   │
└───────────────┘                    └────────────────┘
        │                                     │
        ▼                                     ▼
┌───────────────┐                    ┌────────────────┐
│ proxyconfigmap│                    │ proxydeployment│
│ controller    │                    │ controller     │
│               │                    │                │
│ Generate      │                    │ Create         │
│ mcrouter      │◄───────────────────┤ mcrouter       │
│ config        │                    │ Deployment     │
└───────────────┘                    └────────────────┘
```

### 1.2. Pattern tạo Deployment (proxydeployment/controller.go:279-364)

Code đã có sẵn pattern để tạo Deployment:

```go
func (c *Controller) deploymentForProxy(p *v1alpha1.MemcachedProxy) (*appsv1.Deployment, error) {
    replicas := int32(1)
    d := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            GenerateName:    fmt.Sprintf("%s-mcrouter-", p.Name),
            Namespace:       p.Namespace,
            OwnerReferences: []metav1.OwnerReference{*controller.NewProxyOwnerRef(p)},
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &replicas,
            Selector: &metav1.LabelSelector{...},
            Template: corev1.PodTemplateSpec{
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{{
                        Name:  "mcrouter",
                        Image: p.Spec.McRouter.Image,
                        // ... ports, volumes, security context
                    }},
                },
            },
        },
    }
    return d, nil
}
```

**Pattern này có thể clone để tạo memcached deployment!**

### 1.3. Owner References & Garbage Collection (meta.go:24-39)

Đã có infrastructure để tự động xóa resource:

```go
func NewProxyOwnerRef(c *v1alpha1.MemcachedProxy) *metav1.OwnerReference {
    controller := true
    blockOwnerDeletion := true
    return &metav1.OwnerReference{
        APIVersion:         "ianlewis.org/v1alpha1",
        Kind:               "MemcachedProxy",
        Name:               c.Name,
        UID:                c.UID,
        Controller:         &controller,
        BlockOwnerDeletion: &blockOwnerDeletion,
    }
}
```

**Có thể tái sử dụng cho memcached resources!**

---

## 2. Đề xuất kiến trúc mới

### 2.1. Mở rộng CRD (types.go)

**Thêm field `memcached` vào `MemcachedProxySpec`:**

```go
type MemcachedProxySpec struct {
    Rules      RuleSpec          `json:"rules"`
    McRouter   McRouterSpec      `json:"mcrouter"`

    // ====== TÍnh năng mới ======
    Memcached  *MemcachedSpec    `json:"memcached,omitempty"`  // Optional
}

// MemcachedSpec định nghĩa cấu hình cho memcached instances
type MemcachedSpec struct {
    // Số lượng memcached pods
    Replicas    *int32                     `json:"replicas,omitempty"`

    // Container image
    Image       string                     `json:"image,omitempty"`

    // Memory limit cho memcached (-m flag)
    MemoryLimitMB int32                    `json:"memoryLimitMB,omitempty"`

    // Resource requests/limits
    Resources   corev1.ResourceRequirements `json:"resources,omitempty"`

    // Security context
    SecurityContext *corev1.SecurityContext `json:"securityContext,omitempty"`

    // Port (default 11211)
    Port        *int32                     `json:"port,omitempty"`

    // Extra arguments for memcached
    ExtraArgs   []string                   `json:"extraArgs,omitempty"`
}

func (s *MemcachedSpec) ApplyDefaults() {
    if s.Image == "" {
        s.Image = "memcached:1.6-alpine"
    }
    if s.Replicas == nil {
        replicas := int32(3)
        s.Replicas = &replicas
    }
    if s.Port == nil {
        port := int32(11211)
        s.Port = &port
    }
    if s.MemoryLimitMB == 0 {
        s.MemoryLimitMB = 64
    }
}
```

### 2.2. Thêm controller mới: `memcacheddeployment`

**Tạo file: `internal/controller/memcacheddeployment/controller.go`**

```go
package memcacheddeployment

// Controller tạo và quản lý memcached Deployment
type Controller struct {
    client            clientset.Interface
    ianlewisorgClient ianlewisorgclientset.Interface

    pLister  ianlewisorglisters.MemcachedProxyLister
    dLister  appsv1listers.DeploymentLister
    sLister  corev1listers.ServiceLister

    recorder record.EventRecorder
    queue    workqueue.RateLimitingInterface
    workers  int
    l        *logging.Logger
}

// deploymentForMemcached tạo Deployment cho memcached instances
func (c *Controller) deploymentForMemcached(p *v1alpha1.MemcachedProxy) (*appsv1.Deployment, error) {
    if p.Spec.Memcached == nil {
        return nil, nil  // Không cấu hình memcached => không tạo
    }

    spec := p.Spec.Memcached

    return &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:            fmt.Sprintf("%s-memcached", p.Name),
            Namespace:       p.Namespace,
            OwnerReferences: []metav1.OwnerReference{*controller.NewProxyOwnerRef(p)},
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: spec.Replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{
                    "app":              "memcached",
                    "memcached-proxy":  p.Name,
                },
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{
                        "app":              "memcached",
                        "memcached-proxy":  p.Name,
                    },
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{{
                        Name:  "memcached",
                        Image: spec.Image,
                        Args: []string{
                            "-m", fmt.Sprint(spec.MemoryLimitMB),
                            "-p", fmt.Sprint(*spec.Port),
                            "-vv",  // verbose logging
                        },
                        Ports: []corev1.ContainerPort{{
                            Name:          "memcached",
                            ContainerPort: *spec.Port,
                        }},
                        Resources:       spec.Resources,
                        SecurityContext: spec.SecurityContext,
                    }},
                },
            },
        },
    }, nil
}
```

### 2.3. Thêm controller mới: `memcachedservice`

**Tạo Service để expose memcached pods**

```go
func (c *Controller) serviceForMemcached(p *v1alpha1.MemcachedProxy) (*corev1.Service, error) {
    if p.Spec.Memcached == nil {
        return nil, nil
    }

    return &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:            fmt.Sprintf("%s-memcached", p.Name),
            Namespace:       p.Namespace,
            OwnerReferences: []metav1.OwnerReference{*controller.NewProxyOwnerRef(p)},
        },
        Spec: corev1.ServiceSpec{
            Type: corev1.ServiceTypeClusterIP,
            Selector: map[string]string{
                "app":              "memcached",
                "memcached-proxy":  p.Name,
            },
            Ports: []corev1.ServicePort{{
                Name:       "memcached",
                Port:       *p.Spec.Memcached.Port,
                TargetPort: intstr.FromInt(int(*p.Spec.Memcached.Port)),
            }},
        },
    }, nil
}
```

### 2.4. Update proxyconfigmap controller

**Tự động trỏ proxy đến service của memcached nội bộ**

```go
func (c *Controller) syncHandler(key string) error {
    // ...
    p, err := c.pLister.MemcachedProxies(ns).Get(name)

    // Nếu có spec.Memcached, auto-update rules để trỏ đến service nội bộ
    if p.Spec.Memcached != nil {
        if p.Spec.Rules.Service == nil {
            p.Spec.Rules.Service = &v1alpha1.ServiceSpec{
                Name:      fmt.Sprintf("%s-memcached", p.Name),
                Namespace: p.Namespace,
                Port:      intstr.FromInt(int(*p.Spec.Memcached.Port)),
            }
        }
    }

    // Continue với logic hiện tại...
}
```

### 2.5. Cập nhật main.go

**Đăng ký 2 controllers mới:**

```go
func main() {
    // ... existing code ...

    // ====== CONTROLLER MỚI: Memcached Deployment ======
    m.Register("memcached-deployment", func(ctx *controller.Context) controller.Interface {
        return memcacheddeployment.New(
            "memcached-deployment",
            ctx.Client,
            ianlewisorgClient,
            // ... informers ...
        )
    })

    // ====== CONTROLLER MỚI: Memcached Service ======
    m.Register("memcached-service", func(ctx *controller.Context) controller.Interface {
        return memcachedservice.New(
            "memcached-service",
            ctx.Client,
            ianlewisorgClient,
            // ... informers ...
        )
    })

    // ... existing controllers ...
}
```

---

## 3. Kiến trúc mới - Diagram

```
┌─────────────────────────────────────────────────────────────┐
│              MemcachedProxy CRD (Updated)                    │
│   ┌──────────────────┬────────────┬──────────────────┐     │
│   │ rules            │ mcrouter   │ memcached (NEW)  │     │
│   │ - type           │ - image    │ - replicas       │     │
│   │ - service (opt)  │ - port     │ - image          │     │
│   │ - children       │ - resources│ - memoryLimitMB  │     │
│   └──────────────────┴────────────┴──────────────────┘     │
└─────────────────────────────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌────────────────┐  ┌──────────────────┐
│ proxy         │  │ memcached      │  │ memcachedservice │
│ controller    │  │ deployment     │  │ controller (NEW) │
│               │  │ controller     │  │                  │
│ Apply         │  │ (NEW)          │  │ Create Service   │
│ defaults      │  │                │  │ for memcached    │
│               │  │ Create         │  │ pods             │
└───────────────┘  │ memcached      │  └──────────────────┘
        │          │ Deployment     │           │
        │          └────────────────┘           │
        │                  │                    │
        │                  ▼                    │
        │          ┌────────────────┐           │
        │          │ Memcached Pods │           │
        │          │ (StatefulSet   │           │
        │          │  or Deployment)│           │
        │          └────────────────┘           │
        │                  │                    │
        │                  ├────────────────────┘
        │                  │
        ▼                  ▼
┌───────────────┐  ┌────────────────┐
│ proxyconfigmap│  │ proxyservice   │
│ controller    │  │ controller     │
│               │  │                │
│ Auto-detect   │  │ Create Service │
│ internal      │  │ for mcrouter   │
│ memcached svc │  │                │
└───────────────┘  └────────────────┘
        │                  │
        ▼                  │
┌───────────────┐          │
│ proxydeployment│         │
│ controller    │          │
│               │          │
│ Create        │          │
│ mcrouter      │◄─────────┘
│ Deployment    │
└───────────────┘
        │
        ▼
┌───────────────┐
│ McRouter Pods │
└───────────────┘
```

---

## 4. Ví dụ sử dụng mới

### 4.1. Managed Mode (Operator quản lý toàn bộ)

```yaml
apiVersion: ianlewis.org/v1alpha1
kind: MemcachedProxy
metadata:
  name: my-cache
spec:
  # ====== TÍNH NĂNG MỚI: Operator tự tạo memcached ======
  memcached:
    replicas: 5
    image: memcached:1.6-alpine
    memoryLimitMB: 512
    resources:
      requests:
        cpu: 100m
        memory: 600Mi
      limits:
        cpu: 500m
        memory: 700Mi

  # Proxy sẽ TỰ ĐỘNG trỏ đến service của memcached ở trên
  rules:
    type: "sharded"
    # service: (tự động điền bởi operator)

  mcrouter:
    image: ianmlewis/mcrouter:v0.36.0-2
    port: 11211
```

**Kết quả:** Operator tạo:
- ✅ Memcached Deployment (5 replicas)
- ✅ Memcached Service
- ✅ McRouter ConfigMap (tự động trỏ đến memcached service)
- ✅ McRouter Deployment
- ✅ McRouter Service (cho clients)

### 4.2. Unmanaged Mode (Backward compatible - như hiện tại)

```yaml
apiVersion: ianlewis.org/v1alpha1
kind: MemcachedProxy
metadata:
  name: my-cache
spec:
  # Không có field memcached => user tự quản lý
  rules:
    type: "sharded"
    service:
      name: "my-external-memcached"  # Service do user tạo
      port: 11211

  mcrouter:
    image: ianmlewis/mcrouter:v0.36.0-2
```

**Backward compatible 100%!**

---

## 5. Tính năng đầy đủ sau khi upgrade

| Tính năng | Trước | Sau | Ghi chú |
|-----------|-------|-----|---------|
| **Khởi tạo Memcached** | ❌ | ✅ | Tự động tạo Deployment + Service |
| **Tăng storage** | ❌ | ✅ | Update `memoryLimitMB` trong spec |
| **Scale replicas** | ❌ | ✅ | Update `replicas` trong spec |
| **Xóa Memcached** | ⚠️ | ✅ | Tự động xóa qua OwnerReferences |
| **Monitoring** | ⚠️ | ⚠️→✅ | Có thể thêm Prometheus annotations |
| **Backward compatible** | N/A | ✅ | Mode unmanaged vẫn hoạt động |

---

## 6. Roadmap Implementation

### Phase 1: Foundation (1-2 tuần)
- [ ] Update CRD với `MemcachedSpec`
- [ ] Update codegen (deepcopy, clientset)
- [ ] Tạo `memcacheddeployment` controller (skeleton)
- [ ] Tạo `memcachedservice` controller (skeleton)
- [ ] Unit tests cơ bản

### Phase 2: Core Logic (2-3 tuần)
- [ ] Implement deployment creation logic
- [ ] Implement service creation logic
- [ ] Auto-update rules khi có `spec.memcached`
- [ ] Resource reconciliation (update/delete)
- [ ] Integration tests

### Phase 3: Advanced Features (2-3 tuần)
- [ ] Support scaling (update replicas)
- [ ] Support memory resize (rolling update)
- [ ] Health checks & readiness probes
- [ ] Metrics annotations cho monitoring
- [ ] E2E tests

### Phase 4: Production Ready (1-2 tuần)
- [ ] Documentation
- [ ] Migration guide
- [ ] Helm chart updates
- [ ] RBAC updates
- [ ] Release v0.1.0

**Tổng effort: 6-10 tuần (1 developer)**

---

## 7. Rủi ro và Giải pháp

### Rủi ro 1: Breaking changes
**Giải pháp:** Field `memcached` là optional → backward compatible 100%

### Rủi ro 2: Phức tạp khi debug
**Giải pháp:** Sử dụng Events và Status subresource để expose thông tin

### Rủi ro 3: Conflict với user-managed memcached
**Giải pháp:** Naming convention clear: `{proxyname}-memcached`

### Rủi ro 4: StatefulSet vs Deployment
**Giải pháp:**
- Phase 1: Dùng Deployment (đơn giản)
- Phase 2+: Optional StatefulSet với PersistentVolumes

---

## 8. So sánh với các operator khác

| Feature | Current | After Upgrade | KubeDB | KubeStack |
|---------|---------|---------------|--------|-----------|
| Proxy management | ✅ | ✅ | ❌ | ❌ |
| Memcached lifecycle | ❌ | ✅ | ✅ | ✅ |
| Sharding/Replication | ✅ | ✅ | ⚠️ | ⚠️ |
| StatefulSet support | ❌ | 🔜 | ✅ | ✅ |
| Monitoring | ⚠️ | ✅ | ✅ | ✅ |
| Backup/Restore | ❌ | 🔜 | ✅ | ❌ |

**Sau khi upgrade, operator này sẽ là UNIQUE - vì kết hợp cả proxy và lifecycle!**

---

## 9. Kết luận

### ✅ HOÀN TOÀN KHẢ THI

**Lý do:**
1. **Kiến trúc đã sẵn sàng** - Controller pattern, informers, workqueues đã có
2. **Code reusable** - Pattern tạo Deployment/Service có thể clone
3. **Backward compatible** - Field optional không làm break existing users
4. **Độc đáo** - Kết hợp proxy + lifecycle management

**Effort hợp lý:** 6-10 tuần cho 1 developer có kinh nghiệm Kubernetes

**Giá trị:**
- Giải quyết đầy đủ 4 tính năng bạn yêu cầu
- Tạo ra operator độc nhất (proxy + lifecycle)
- Tối ưu user experience (single CRD cho all-in-one)

---

## 10. Next Steps

Nếu muốn tiến hành:

1. **Fork repository** và tạo branch `feature/managed-memcached`
2. **Update CRD** với MemcachedSpec
3. **Tạo POC** cho memcacheddeployment controller
4. **Test thủ công** với minikube/kind
5. **Iterate** dựa trên feedback

Bạn có muốn tôi tạo POC code cho Phase 1 không?
