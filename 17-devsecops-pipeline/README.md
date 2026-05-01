# 项目十七: DevSecOps 安全左移流水线

## 一、项目背景

传统安全实践的问题：
- 安全扫描在发布前一刻才执行，发现漏洞时修复成本极高
- 开发团队与安全团队互相推诿责任
- 合规检查依赖人工审计，效率低且易遗漏
- 供应链攻击（第三方组件漏洞）难以管控

DevSecOps **安全左移**核心思想：将安全检查嵌入 CI/CD 流水线的每个环节，实现"谁开发谁负责安全"。

## 二、架构设计

### 2.1 安全流水线架构图

```
┌─ Code Commit ──► Build ──► Unit Test ──► Scan ──► Package ──► Deploy ──► Monitor
│                  │         │             │          │           │          │
│ GitHook          │ Pre-    │ SAST        │ Image    │ HCL/K8s  │ Runtime  │
│ (AuthN)          │ commit  │ SonarQube   │ Trivy    │ OPA/Kyverno│ Falco  │
│                  │ Semgrep │ ESLint/     │ Harbor   │ Gatekeeper │ Syscall│
│                  │         │ Checkstyle  │ Scanner  │            │ Audit  │
└──────────────────┴─────────┴─────────────┴──────────┴──────────┴──────────┘
                        │              │                    │
                    ┌───▼───┐      ┌──▼───┐           ┌────▼───┐
                    │Secret │      │SCA   │           │Config  │
                    │Scan   │      │Dependency│        │Drift  │
                    │Gitleak│      │Retire.js│        │Checker│
                    └───────┘      └────────┘           └──────┘
```

### 2.2 工具链选型

| 阶段 | 开源方案 | 阿里云 | AWS | 华为云 | 腾讯云 |
|------|----------|--------|-----|--------|--------|
| SAST | SonarQube/Semgrep | SCA(代码分析) | CodeGuru | HiAgent | 代码审计平台 |
| 镜像扫描 | Trivy/Aqua | 镜像安全扫描 | Inspector ECR | 容器镜像扫描 | 镜像安全扫描 |
| 基础设施扫描 | Checkov/TfSec | ROS模板验证 | CloudFormation Guard | CTF模板检查 | Terraform审计 |
| K8s合规 | kube-bench/Kyverno | ACK安全中心 | Security Hub | 主机安全 | 容器安全中心 |
| 运行时检测 | Falco/Wazuh | 威胁情报 | GuardDuty | 安全态势 | 安骑士 |

## 三、完整 CI/CD 流水线

### 3.1 GitLab CI/CD - 全栈安全流水线

```yaml
# .gitlab-ci.yml
stages:
  - pre-commit
  - build
  - test
  - security-scan
  - package
  - deploy-staging
  - security-validate
  - deploy-prod

variables:
  DOCKER_REGISTRY: registry.cn-hangzhou.aliyuncs.com/myorg
  IMAGE_NAME: product-api
  TRIVY_FORMAT: table

# ==================== 阶段1: Pre-commit ====================
pre-commit-security:
  stage: pre-commit
  image: python:3.11-slim
  rules:
    - changes:
        - "**/*.py"
        - "**/*.js"
        - "**/*.go"
  script:
    # Secret 泄露检测
    - pip install gitleaks --quiet
    - gitleaks detect --source=. --report-format=json --report-path=gitleaks-report.json
    # 静态代码扫描
    - pip install bandit --quiet
    - bandit -r src/ -f json -o bandit-report.json || true
    artifacts:
      reports:
        secret_detection: [gitleaks-report.json]
        sast: [bandit-report.json]
  allow_failure: true

# ==================== 阶段2: Build ====================
build-application:
  stage: build
  image: docker:24-dind
  services:
    - docker:24-docker
  script:
    - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USER" --password-stdin
    - docker build --target builder -t $DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA ./src
    - docker tag $DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA $DOCKER_REGISTRY/$IMAGE_NAME:$CI_PIPELINE_ID
  artifacts:
    paths:
      - src/Dockerfile
      - src/requirements.txt

# ==================== 阶段3: Unit Test ====================
run-tests:
  stage: test
  image: python:3.11
  dependencies:
    - build-application
  script:
    - cd src
    - pip install -r requirements.txt
    - pytest tests/ --junitxml=test-results.xml -v
    - coverage report --fail-under=80
  artifacts:
    reports:
      junit: test-results.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

# ==================== 阶段4: 多维安全扫描 ====================
image-vulnerability-scan:
  stage: security-scan
  image: aquasec/trivy:latest
  needs:
    - build-application
  script:
    # 镜像层漏洞扫描
    - trivy image \
        --severity HIGH,CRITICAL \
        --exit-code 1 \
        --format table \
        --timeout 5m \
        $DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA
    # SBOM 生成
    - trivy image \
        --format spdx-json \
        --output sbom-${CI_COMMIT_SHA}.spdx.json \
        $DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA
    # 文件系统配置检查
    - trivy config --scanners configs \
        --format sarif \
        --output trivy-config.sarif \
        src/
  artifacts:
    reports:
      sast: [trivy-config.sarif]
    paths:
      - sbom-${CI_COMMIT_SHA}.spdx.json

dependency-scanning:
  stage: security-scan
  image: node:20-alpine
  needs:
    - build-application
  script:
    - cd src
    - npm audit --audit-level=high --json > npm-audit-report.json || true
    # 替换为 NPM audit 的 JSON 输出格式适配
    - |
      if grep -q '"vulnerabilities"' npm-audit-report.json; then
        VULNS=$(cat npm-audit-report.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d.get('vulnerabilities',{})))")
        if [ "$VULNS" != "0" ]; then
          echo "FAIL: Found $VULNS vulnerabilities"; exit 1
        fi
      fi
  allow_failure: false
  artifacts:
    reports:
      dependency_scanning: [npm-audit-report.json]

secret-detection-post-build:
  stage: security-scan
  image: zricethezav/gitleaks:latest
  needs:
    - build-application
  script:
    - gitleaks protect --source=. --verbose --redact
    - echo "✅ No secrets detected in repository"
  allow_failure: false

# ==================== 阶段5: 打包到私有仓库 ====================
package-and-push:
  stage: package
  image: docker:24-dind
  needs:
    - run-tests
    - image-vulnerability-scan
    - dependency-scanning
  variables:
    TRIVY_IGNORE_UNFIXED: "true"
  script:
    - |
      # 仅推送无高危漏洞的镜像
      IMAGE_TAG="$DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA"
      docker push $IMAGE_TAG
      docker tag $IMAGE_TAG $DOCKER_REGISTRY/$IMAGE_NAME:latest
      docker push $DOCKER_REGISTRY/$IMAGE_NAME:latest
    # Harbor API 同步元数据
    - |
      curl -k -X POST "https://harbor.example.com/api/v2.0/projects/${HARBOR_PROJECT}/repositories/${IMAGE_NAME}/artifacts/${CI_COMMIT_SHA}:updateImageStorageQuota" \
        -H "Authorization: Bearer ${HARBOR_TOKEN}"
  only:
    - main
    - release/*

# ==================== 阶段6: 部署到预发环境 ====================
deploy-staging:
  stage: deploy-staging
  image: bitnami/kubectl:latest
  needs:
    - package-and-push
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl set image deployment/product-api \
        product-api=$DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA \
        -n staging
    - kubectl rollout status deployment/product-api -n staging --timeout=300s

# ==================== 阶段7: 部署前安全验证 ====================
security-validation:
  stage: security-validate
  image: python:3.11
  needs:
    - deploy-staging
  before_script:
    - pip install checkov
  script:
    # IaC 合规检查
    - checkov -d k8s-manifests/ --framework kubernetes --compact
    - checkov -d terraform-envs/staging/ --compact
    # Kubernetes 安全基线检查
    - kubectl run kube-bench --restart=Never --image=aquasec/kube-bench:latest \
        -n default -- --targets=pod --quiet || true
  allow_failure: true

# ==================== 阶段8: 生产环境部署（审批制）====================
deploy-production:
  stage: deploy-prod
  image: bitnami/kubectl:latest
  needs:
    - security-validation
  environment:
    name: production
    url: https://www.example.com
  script:
    # 金丝雀发布
    - kubectl set image deployment/product-api \
        product-api=$DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA \
        -n production
    # 等待健康检查通过后升级
    - kubectl wait --for=condition=available deployment/product-api -n production --timeout=120s
    # 逐步扩量
    - kubectl scale deployment/product-api --replicas=3 -n production
  when: manual
  resources_group: deploy-prod-critical

# ==================== 告警通知 ====================
notify-slack:
  stage: .post
  image: alpine/curl
  needs:
    - deploy-production
  script:
    - |
      STATUS=$PIPELINE_STATUS
      curl -X POST "$SLACK_WEBHOOK" \
        -H 'Content-type: application/json' \
        -d "{\"text\":\"[DevSecOps] Pipeline #$CI_PIPELINE_ID $STATUS: $CI_PROJECT_NAME\"}"
  rules:
    - if: '$CI_PIPELINE_STATUS == "failed"'
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_STATUS == "success"'
```

### 3.2 OPA Policy 门禁策略

```rego
# policy/restrict-latest-tag.rego
package kubernetes.admission

deny[msg] {
  input.request.object.spec.template.spec.containers[_].image.tags[_] == "latest"
  msg := sprintf("Image '%s' must not use 'latest' tag", [input.request.object.spec.template.spec.containers[_].image])
}

# 禁止使用 privileged 模式
deny[msg] {
  c := input.request.object.spec.template.spec.containers[_]
  c.securityContext.privileged == true
  msg := sprintf("Container '%s' must not run in privileged mode", [c.name])
}

# 要求资源限制
deny[msg] {
  c := input.request.object.spec.template.spec.containers[_]
  not c.resources.limits.memory
  msg := sprintf("Container '%s' must define memory limits", [c.name])
}

deny[msg] {
  c := input.request.object.spec.template.spec.containers[_]
  not c.resources.limits.cpu
  msg := sprintf("Container '%s' must define CPU limits", [c.name])
}

# 禁止 root 用户运行
deny[msg] {
  c := input.request.object.spec.template.spec.containers[_]
  not c.securityContext.runAsNonRoot
  msg := sprintf("Container '%s' must set runAsNonRoot=true", [c.name])
}

# 只能从白名单仓库拉取镜像
whitelist_registries = ["registry.cn-hangzhou.aliyuncs.com/myorg", "docker.io/library"]
deny[msg] {
  img := input.request.object.spec.template.spec.containers[_].image
  not startswith(img.registry, whitelist_registries[_])
  msg := sprintf("Registry '%s' not in approved list", [img.registry])
}
```

### 3.3 Kyverno Policy (K8s Native)

```yaml
# kyverno-policy-enforce-resources.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
  annotations:
    policies.kyverno.io/title: Require Resource Limits
    policies.kyverno.io/category: Best Practices
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: limit-memory-cpu
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "CPU and memory resource limits are required."
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"
---
# 自动添加默认资源限制
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-resources
spec:
  rules:
  - name: inject-default-resources
    match:
      any:
      - resources:
          kinds: ["Pod"]
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - (name): "*"
            resources:
              limits:
                memory: {{`.Values.defaultMemoryLimit | default "256Mi"`}}
                cpu: {{`.Values.defaultCpuLimit | default "500m"`}}
              requests:
                memory: {{`.Values.defaultMemoryRequest | default "128Mi"`}}
                cpu: {{`.Values.defaultCpuRequest | default "100m"`}}
```

### 3.4 Harbor 仓库安全配置

```bash
# Harbor 高级安全设置
#!/bin/bash
# harborg-config.sh

# 1. 启用内容信任（Docker Notary）
# Harbor Admin Console → System Settings → Content Trust → Enabled

# 2. 配置自动漏洞扫描
curl -X PUT "https://harbor.example.com/api/v2.0/configurations" \
  -H "Authorization: Bearer $(harbor-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "project_creation_restriction": "admin-only",
    "self_registration": "false",
    "scan_image_push": "true",
    "scanner_verification": "true",
    "storage_quota_per_project": 107374182400
  }'

# 3. 创建项目并绑定扫描器
harbor-create-project() {
  local PROJECT_NAME=$1
  curl -X POST "https://harbor.example.com/api/v2.0/projects" \
    -H "Authorization: Bearer $(harbor-token)" \
    -H "Content-Type: application/json" \
    -d "{
      \"project_name\": \"$PROJECT_NAME\",
      \"public\": false,
      \"reuse_public_cid\": false
    }"
}

# 4. 设置扫描调度
# Trivy 每 12h 增量扫描新镜像
crontab <<EOF
0 */12 * * * /usr/local/bin/harbor-sync-scanner-metadata.sh
EOF
```

## 四、多云对比与适配

### 4.1 各云平台原生安全服务对比

| 功能 | 阿里云 | AWS | 华为云 | 腾讯云 | 天翼云 |
|------|--------|-----|--------|--------|--------|
| 代码审计 | SCA | CodeGuru | 代码审计 | 云原生应用治理平台 | - |
| 镜像扫描 | 容器镜像扫描 | Inspector | 镜像安全管理 | 容器安全中心 | - |
| IaC审计 | ROS模板验证 | CloudShell + Checkov | CTF验证 | 轻量防火墙 | - |
| K8s安全 | ACK安全中心 | EKS Access Analyzer | CCE安全组 | TKE安全中心 | - |
| WAF | Web应用防火墙 | WAF | 企业主机安全 | 云WAF | - |

## 五、运维管理

### 5.1 日常巡检清单

```bash
# 1. 最近一次扫描结果汇总
echo "=== 漏洞统计 ==="
cat trivy-report.json | jq '[.Results[].Vulnerabilities[] | select(.Severity=="HIGH" or .Severity=="CRITICAL")] | length'

# 2. Gitleaks 历史趋势
gitleaks log --report-format=json 2>/dev/null | \
  jq '[. [] | .RuleID] | group_by(.) | map({rule: .[0], count: length}) | sort_by(-.count)'

# 3. Harbor 仓库空间占用
curl -s "https://harbor.example.com/api/v2.0/statistics" | jq '.storage_human'
```

### 5.2 故障排查指南

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| SAST 误报率高 | SonarQube 规则太严 | 调整 Quality Gate 阈值，忽略已知假阳性 |
| Trivy 扫描慢 | DB 未定期更新 | `trivy image --clear-cache && trivy image-db pull` |
| OPA 策略阻塞部署 | 策略过于严格 | 使用 `Audit` 而非 `Enforce` 模式先观察 |
| 镜像推送到非白名单 | 本地 Docker tag 错误 | 构建时使用 FQDN tag |

## 六、成本估算

| 组件 | 规格 | 月费用估算 |
|------|------|-----------|
| GitLab Runner | 4x4C8G | ¥800/月 |
| SonarQube | 8C16G + DB | ¥600/月 |
| Harbor | 4C8G + 200GB | ¥400/月 |
| Trivy | 免费开源 | ¥0 |
| Kyverno | 免费开源(K8s Controller) | 自付K8s资源 |
| 阿里云 SCA | 企业版 | ¥2,000/月 |

## 七、面试考点

1. **安全左移理念**: 为什么越早发现漏洞越便宜？(Shift Left ≈ 修复成本指数级下降)
2. **OPA vs Kyverno**: OPA 基于 Rego 通用策略引擎；Kyverno 是 K8s-native 策略控制器
3. **镜像签名与验证**: cosign/distroless + keyless signing + admission webhook 验证
4. **SBOM**: Software Bill of Materials， SPDX/CycloneDX 格式，用于供应链溯源
5. **零信任网络**: 不信任任何内部流量，所有 service-to-service 通信需 mTLS + AuthorizationPolicy

## 八、课后练习

1. 搭建完整的 GitLab CI/CD + SonarQube + Trivy 安全流水线
2. 编写 5 条 Kyverno Policy 强制资源限制和 RunAsNonRoot
3. 对已有 Kubernetes 集群运行 kube-bench CIS Benchmark 检查
4. 使用 cosign 为镜像签名并在 admission webhook 中验证签名
