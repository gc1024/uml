# GitLab CICD 前端自动化部署方案

> 基于 GitLab 私有化部署的最优实施方案

---

## 一、架构设计

### 1.1 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         GitLab 私有部署                          │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  develop │  │ feature/*│  │   main   │  │   tags   │       │
│  │  分支    │  │  分支    │  │  分支    │  │  (v1.*)  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │             │             │             │               │
│       └──────────┬──┴─────────────┴─────────────┘               │
│                  ▼                                              │
│         ┌──────────────────┐                                    │
│         │ GitLab CI/CD     │                                    │
│         │ Pipeline         │                                    │
│         └────────┬─────────┘                                    │
└──────────────────┼──────────────────────────────────────────────┘
                   │ Webhook / SSH
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      部署服务器                                   │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     部署脚本层                             │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │  │
│  │  │deploy.sh │ │build.sh  │ │health.sh │ │rollback  │    │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     资源管理层                             │  │
│  │  ┌────────────────┐    ┌────────────────┐                │  │
│  │  │   releases/    │    │  versions.json │                │  │
│  │  │  ├─ v1.2.3/    │    │  ├─ current   │                │  │
│  │  │  ├─ v1.2.4/    │    │  ├─ previous  │                │  │
│  │  │  └─ v1.2.5/    │    │  └─ history[] │                │  │
│  │  └────────────────┘    └────────────────┘                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     服务层                                 │  │
│  │  ┌────────────────┐              ┌────────────────┐       │  │
│  │  │   Nginx        │              │   OSS Upload   │       │  │
│  │  │  current ->    │─────────────>│   (async)      │       │  │
│  │  │  releases/v1.2.5│              └────────────────┘       │  │
│  │  └────────────────┘                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                       飞书                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  部署通知  │  验证提醒  │  失败告警  │  回滚通知             │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 目录结构设计

```
# 部署服务器目录结构
/opt/frontend-deploy/
├── scripts/                          # 部署脚本
│   ├── deploy.sh                     # 主部署脚本
│   ├── build.sh                      # 构建脚本
│   ├── health-check.sh               # 健康检查脚本
│   ├── rollback.sh                   # 回滚脚本
│   ├── notify.sh                     # 飞书通知脚本
│   └── lib/                          # 公共库
│       ├── logger.sh                 # 日志函数
│       ├── version.sh                # 版本管理函数
│       └── oss.sh                    # OSS 操作函数
├── configs/                          # 配置文件
│   ├── deploy.conf                   # 部署配置
│   └── projects.conf                 # 项目配置
├── templates/                        # 模板文件
│   └── nginx.conf.template           # Nginx 配置模板
├── logs/                             # 日志目录
│   ├── deploy.log                    # 部署日志
│   ├── error.log                     # 错误日志
│   └── audit.log                     # 审计日志
└── temp/                             # 临时目录

# 应用部署目录结构
/var/www/frontend/
├── current -> releases/app-20240429-163000  # 软链接，指向当前版本
├── previous -> releases/app-20240429-153000 # 软链接，指向上一个版本
├── releases/                                 # 版本目录
│   ├── app-20240429-143000/
│   │   ├── dist/                           # 构建产物
│   │   ├── meta.json                       # 版本元数据
│   │   └── README.md                       # 部署说明
│   ├── app-20240429-153000/
│   └── app-20240429-163000/
└── versions.json                             # 版本索引文件
```

---

## 二、GitLab CI/CD 配置

### 2.1 .gitlab-ci.yml 完整配置

```yaml
# .gitlab-ci.yml
stages:
  - build
  - deploy-test
  - deploy-prod

variables:
  # 构建配置
  NODE_VERSION: "18"
  CACHE_KEY_PREFIX: "frontend"
  # 部署服务器配置
  DEPLOY_SERVER: "deploy@your-server.com"
  DEPLOY_PATH: "/opt/frontend-deploy"
  # 飜飞书配置
  FEISHU_WEBHOOK_URL: "${FEISHU_WEBHOOK}"

# 缓存配置
cache:
  key: ${CACHE_KEY_PREFIX}-${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/

# 构建阶段
build:frontend:
  stage: build
  image: node:${NODE_VERSION}
  script:
    - echo "构建前端项目..."
    - npm ci --cache .npm --prefer-offline
    - npm run build
    - echo "构建完成"
  artifacts:
    paths:
      - dist/
    expire_in: 1 day
  only:
    - develop
    - main
    - /^release\/.*$/
    - /^v\d+\.\d+\.\d+$/

# 测试环境部署（手动触发）
deploy:test:
  stage: deploy-test
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client bash
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
  script:
    - echo "部署到测试环境..."
    - |
      ssh $DEPLOY_SERVER "$DEPLOY_PATH/scripts/deploy.sh \
        --env=test \
        --branch=$CI_COMMIT_REF_NAME \
        --commit=$CI_COMMIT_SHA \
        --triggerer=$GITLAB_USER_NAME \
        --pipeline=$CI_PIPELINE_ID"
  when: manual  # 手动触发
  only:
    - develop
    - /^feature\/.*$/
  environment:
    name: test
    url: http://test.example.com
    on_stop: cleanup:test

# 生产环境部署（Tag 触发）
deploy:production:
  stage: deploy-prod
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client bash
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
  script:
    - echo "部署到生产环境..."
    - |
      ssh $DEPLOY_SERVER "$DEPLOY_PATH/scripts/deploy.sh \
        --env=production \
        --tag=$CI_COMMIT_TAG \
        --commit=$CI_COMMIT_SHA \
        --triggerer=$GITLAB_USER_NAME \
        --pipeline=$CI_PIPELINE_ID"
  only:
    - /^v\d+\.\d+\.\d+$/
  environment:
    name: production
    url: https://www.example.com

# 清理测试环境旧版本
cleanup:test:
  stage: deploy-test
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client bash
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
  script:
    - echo "清理测试环境旧版本..."
    - ssh $DEPLOY_SERVER "$DEPLOY_PATH/scripts/cleanup.sh --env=test"
  when: manual
  only:
    - develop
    - /^feature\/.*$/
```

### 2.2 GitLab CI/CD 变量配置

在 GitLab 项目设置 > CI/CD > Variables 中配置：

| 变量名 | 类型 | 保护 | 描述 |
|--------|------|------|------|
| `SSH_PRIVATE_KEY` | File | No | SSH 私钥，用于连接部署服务器 |
| `SSH_KNOWN_HOSTS` | Variable | No | SSH known_hosts 内容 |
| `FEISHU_WEBHOOK` | Variable | No | 飞书机器人 Webhook URL |

---

## 三、部署脚本实现

### 3.1 主部署脚本 deploy.sh

```bash
#!/bin/bash
######################################################################
# 部署脚本 - 支持测试环境和生产环境
# 用法: ./deploy.sh --env=<test|production> --branch=<name> --tag=<tag>
######################################################################

set -euo pipefail

# 脚本目录
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"

# 加载公共库
source "${SCRIPT_DIR}/lib/logger.sh"
source "${SCRIPT_DIR}/lib/version.sh"
source "${SCRIPT_DIR}/lib/oss.sh"
source "${PROJECT_ROOT}/configs/deploy.conf"

# 全局变量
ENV=""
BRANCH=""
TAG=""
COMMIT=""
TRIGGERER=""
PIPELINE_ID=""
VERSION=""
DEPLOY_TIMESTAMP=""

######################################################################
# 函数定义
######################################################################

# 显示帮助信息
show_help() {
    cat << EOF
用法: $(basename "$0") [选项]

选项:
  --env=<test|production>    部署环境（必需）
  --branch=<name>           Git 分支名（测试环境必需）
  --tag=<tag>               Git 标签（生产环境必需）
  --commit=<sha>            提交哈希
  --triggerer=<name>        触发者名称
  --pipeline=<id>           Pipeline ID
  --rollback                回滚模式
  -h, --help                显示帮助信息

示例:
  # 测试环境部署
  $(basename "$0") --env=test --branch=develop --commit=abc123 --triggerer="张三"

  # 生产环境部署
  $(basename "$0") --env=production --tag=v1.2.3 --commit=abc123 --triggerer="李四"

  # 回滚
  $(basename "$0") --env=production --rollback
EOF
}

# 解析命令行参数
parse_args() {
    while [[ $# -gt 0 ]]; do
        case $1 in
            --env=*)
                ENV="${1#*=}"
                shift
                ;;
            --branch=*)
                BRANCH="${1#*=}"
                shift
                ;;
            --tag=*)
                TAG="${1#*=}"
                shift
                ;;
            --commit=*)
                COMMIT="${1#*=}"
                shift
                ;;
            --triggerer=*)
                TRIGGERER="${1#*=}"
                shift
                ;;
            --pipeline=*)
                PIPELINE_ID="${1#*=}"
                shift
                ;;
            --rollback)
                ROLLBACK_MODE=true
                shift
                ;;
            -h|--help)
                show_help
                exit 0
                ;;
            *)
                error "未知参数: $1"
                show_help
                exit 1
                ;;
        esac
    done

    # 参数验证
    if [[ -z "${ENV}" ]]; then
        error "必须指定 --env 参数"
        exit 1
    fi

    if [[ "${ENV}" == "production" && -z "${TAG}" ]]; then
        error "生产环境必须指定 --tag 参数"
        exit 1
    fi

    if [[ "${ENV}" == "test" && -z "${BRANCH}" ]]; then
        error "测试环境必须指定 --branch 参数"
        exit 1
    fi
}

# 获取环境配置
get_env_config() {
    case "${ENV}" in
        test)
            PROJECT_NAME="${TEST_PROJECT_NAME}"
            DEPLOY_BASE_DIR="${TEST_DEPLOY_BASE_DIR}"
            DEPLOY_URL="${TEST_DEPLOY_URL}"
            HEALTH_CHECK_URL="${TEST_HEALTH_CHECK_URL}"
            KEEP_RELEASES="${TEST_KEEP_RELEASES}"
            ;;
        production)
            PROJECT_NAME="${PROD_PROJECT_NAME}"
            DEPLOY_BASE_DIR="${PROD_DEPLOY_BASE_DIR}"
            DEPLOY_URL="${PROD_DEPLOY_URL}"
            HEALTH_CHECK_URL="${PROD_HEALTH_CHECK_URL}"
            KEEP_RELEASES="${PROD_KEEP_RELEASES}"
            ;;
        *)
            error "未知的环境: ${ENV}"
            exit 1
            ;;
    esac
}

# 初始化部署
init_deploy() {
    info "开始初始化部署..."
    DEPLOY_TIMESTAMP=$(date +%Y%m%d-%H%M%S)

    # 生成版本号
    if [[ "${ENV}" == "production" ]]; then
        VERSION="${TAG}"
    else
        VERSION="app-${DEPLOY_TIMESTAMP}"
    fi

    info "环境: ${ENV}"
    info "版本: ${VERSION}"
    info "分支: ${BRANCH:-${TAG}}"
    info "提交: ${COMMIT:-unknown}"
    info "触发者: ${TRIGGERER:-unknown}"
    info "Pipeline: ${PIPELINE_ID:-unknown}"

    # 创建必要的目录
    local release_dir="${DEPLOY_BASE_DIR}/releases/${VERSION}"
    mkdir -p "${release_dir}"
    mkdir -p "${DEPLOY_BASE_DIR}/temp"
    mkdir -p "${DEPLOY_BASE_DIR}/logs"

    # 记录部署开始
    log_audit "deploy_start" "version=${VERSION}, env=${ENV}, triggerer=${TRIGGERER}"
}

# 拉取代码
clone_code() {
    info "开始拉取代码..."

    local temp_dir="${DEPLOY_BASE_DIR}/temp/${VERSION}"
    rm -rf "${temp_dir}"
    mkdir -p "${temp_dir}"

    # 生产环境从 tag 拉取，测试环境从分支拉取
    if [[ "${ENV}" == "production" ]]; then
        git clone --depth 1 --branch "${TAG}" "${GIT_REPO_URL}" "${temp_dir}"
    else
        git clone --depth 1 --branch "${BRANCH}" "${GIT_REPO_URL}" "${temp_dir}"
    fi

    cd "${temp_dir}"
    git checkout "${COMMIT:-HEAD}"

    info "代码拉取完成: $(git log -1 --format='%h - %s')"
}

# 构建应用
build_app() {
    info "开始构建应用..."

    local temp_dir="${DEPLOY_BASE_DIR}/temp/${VERSION}"

    cd "${temp_dir}"

    # 安装依赖
    info "安装依赖..."
    npm ci --prefer-offline --no-audit

    # 构建
    info "执行构建..."
    npm run build

    # 移动构建产物
    local release_dir="${DEPLOY_BASE_DIR}/releases/${VERSION}"
    mv "${temp_dir}/dist" "${release_dir}/"

    # 清理临时目录
    rm -rf "${temp_dir}"

    info "构建完成"
}

# 生成版本元数据
generate_metadata() {
    info "生成版本元数据..."

    local release_dir="${DEPLOY_BASE_DIR}/releases/${VERSION}"
    local meta_file="${release_dir}/meta.json"

    cat > "${meta_file}" << EOF
{
  "version": "${VERSION}",
  "environment": "${ENV}",
  "branch": "${BRANCH:-${TAG}}",
  "commit": "${COMMIT:-unknown}",
  "commit_message": "$(git log -1 --pretty=%B ${COMMIT:-HEAD} 2>/dev/null || echo 'unknown')",
  "deployTime": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "deployTimestamp": "${DEPLOY_TIMESTAMP}",
  "triggerer": "${TRIGGERER:-unknown}",
  "pipelineId": "${PIPELINE_ID:-unknown}",
  "buildNode": "$(node -v)",
  "buildNpm": "$(npm -v)"
}
EOF

    info "元数据已生成: ${meta_file}"
}

# 更新版本索引文件
update_versions_json() {
    info "更新版本索引..."

    local versions_file="${DEPLOY_BASE_DIR}/versions.json"
    local current_version=$(get_current_version)
    local release_dir="${DEPLOY_BASE_DIR}/releases/${VERSION}"
    local meta_file="${release_dir}/meta.json"

    # 读取元数据
    local version_info=$(cat "${meta_file}")

    # 更新 versions.json
    if [[ -f "${versions_file}" ]]; then
        # 读取当前内容
        local current_json=$(cat "${versions_file}")

        # 使用 jq 更新（如果没有 jq，使用 sed）
        if command -v jq &> /dev/null; then
            echo "${current_json}" | jq \
                --arg current "${VERSION}" \
                --arg previous "${current_version:-null}" \
                --argjson info "${version_info}" \
                '.current = $current | .previous = $previous | .history = [$info] + .history | .history = .history[0:10]' \
                > "${versions_file}.tmp"
            mv "${versions_file}.tmp" "${versions_file}"
        else
            # 备用方案：使用 Python
            python3 << EOF
import json

with open('${versions_file}', 'r') as f:
    data = json.load(f)

data['current'] = '${VERSION}'
data['previous'] = '${current_version}'
info = ${version_info}
data['history'].insert(0, info)
data['history'] = data['history'][:10]

with open('${versions_file}', 'w') as f:
    json.dump(data, f, indent=2, ensure_ascii=False)
EOF
        fi
    else
        # 创建新文件
        cat > "${versions_file}" << EOF
{
  "current": "${VERSION}",
  "previous": "${current_version}",
  "history": [
${version_info}
  ]
}
EOF
    fi

    info "版本索引已更新: ${versions_file}"
}

# 部署应用
deploy_app() {
    info "开始部署应用..."

    local release_dir="${DEPLOY_BASE_DIR}/releases/${VERSION}"

    # 更新 current 软链接
    ln -sfn "${release_dir}" "${DEPLOY_BASE_DIR}/current"

    # 更新 previous 软链接
    local current_version=$(get_current_version)
    if [[ -n "${current_version}" && "${current_version}" != "${VERSION}" ]]; then
        local previous_dir="${DEPLOY_BASE_DIR}/releases/${current_version}"
        if [[ -d "${previous_dir}" ]]; then
            ln -sfn "${previous_dir}" "${DEPLOY_BASE_DIR}/previous"
        fi
    fi

    # 设置权限
    chown -h "${NGINX_USER}:${NGINX_USER}" "${DEPLOY_BASE_DIR}/current"
    chown -R "${NGINX_USER}:${NGINX_USER}" "${release_dir}"

    info "应用部署完成"
}

# 重启 Nginx
reload_nginx() {
    info "重载 Nginx 配置..."

    # 测试配置
    if ! sudo nginx -t 2>&1; then
        error "Nginx 配置测试失败"
        return 1
    fi

    # 优雅重载
    sudo nginx -s reload

    info "Nginx 重载完成"
}

# 健康检查
health_check() {
    info "开始健康检查..."

    source "${SCRIPT_DIR}/health-check.sh"

    local max_retries=${HEALTH_CHECK_MAX_RETRIES:-30}
    local retry_interval=${HEALTH_CHECK_INTERVAL:-2}

    for i in $(seq 1 ${max_retries}); do
        info "健康检查第 ${i}/${max_retries} 次..."

        if check_health "${HEALTH_CHECK_URL}"; then
            info "✓ 健康检查通过"
            return 0
        fi

        sleep ${retry_interval}
    done

    error "✗ 健康检查失败"
    return 1
}

# 上传到 OSS
upload_to_oss() {
    info "开始上传到 OSS..."

    local release_dir="${DEPLOY_BASE_DIR}/releases/${VERSION}"

    # 异步上传
    upload_oss_async "${release_dir}" "${ENV}" "${VERSION}" &

    info "OSS 上传任务已启动（后台运行）"
}

# 清理旧版本
cleanup_old_releases() {
    info "清理旧版本..."

    local keep_count=${KEEP_RELEASES}
    local releases_dir="${DEPLOY_BASE_DIR}/releases"

    # 获取所有版本（按时间倒序）
    local versions=($(ls -1t "${releases_dir}" | grep -E "^app-[0-9]{8}-[0-9]{6}$|^v[0-9]+\.[0-9]+\.[0-9]+$" || true))

    # 保留最新的 N 个版本
    if [[ ${#versions[@]} -gt ${keep_count} ]]; then
        local to_delete=("${versions[@]:${keep_count}}")

        for version in "${to_delete[@]}"; do
            info "删除旧版本: ${version}"
            rm -rf "${releases_dir}/${version}"
        done
    fi

    info "清理完成"
}

# 发送成功通知
notify_success() {
    info "发送成功通知..."

    local message="✅ **部署成功**

**环境:** ${ENV}
**版本:** ${VERSION}
**分支:** ${BRANCH:-${TAG}}
**提交:** ${COMMIT:-unknown}
**触发者:** ${TRIGGERER:-unknown}
**访问地址:** ${DEPLOY_URL}
**部署时间:** $(date '+%Y-%m-%d %H:%M:%S')

请访问 ${DEPLOY_URL} 进行验证"

    send_feishu_notification "${message}"
}

# 发送失败通知
notify_failure() {
    local error_msg="$1"

    local message="❌ **部署失败**

**环境:** ${ENV}
**版本:** ${VERSION}
**错误:** ${error_msg}
**触发者:** ${TRIGGERER:-unknown}

@${TRIGGERER:-所有人} 请立即处理！"

    send_feishu_notification "${message}" true
}

# 发送验证提醒
notify_verify() {
    local message="🔍 **部署完成，请验证**

**环境:** ${ENV}
**版本:** ${VERSION}
**访问地址:** ${DEPLOY_URL}
**部署时间:** $(date '+%Y-%m-%d %H:%M:%S')

@${TRIGGERER:-所有人} 请点击链接验证部署结果"

    send_feishu_notification "${message}"
}

# 记录审计日志
log_audit() {
    local event="$1"
    local details="$2"

    local audit_file="${PROJECT_ROOT}/logs/audit.log"
    local timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)

    echo "{\"timestamp\":\"${timestamp}\",\"event\":\"${event}\",\"details\":\"${details}\"}" >> "${audit_file}"
}

# 主流程
main() {
    parse_args "$@"
    get_env_config

    if [[ "${ROLLBACK_MODE:-false}" == "true" ]]; then
        # 回滚模式
        source "${SCRIPT_DIR}/rollback.sh"
        rollback "${ENV}"
        exit $?
    fi

    # 正常部署流程
    init_deploy
    notify_start

    if clone_code && build_app && generate_metadata; then
        update_versions_json
        deploy_app
        upload_to_oss
        reload_nginx

        if health_check; then
            cleanup_old_releases
            notify_success
            notify_verify
            log_audit "deploy_success" "version=${VERSION}, env=${ENV}"
            exit 0
        else
            # 健康检查失败，自动回滚
            error "健康检查失败，开始自动回滚..."
            source "${SCRIPT_DIR}/rollback.sh"
            rollback "${ENV}"
            notify_failure "健康检查失败，已自动回滚"
            log_audit "deploy_rollback" "version=${VERSION}, env=${ENV}, reason=health_check_failed"
            exit 1
        fi
    else
        # 部署失败
        notify_failure "构建或部署失败"
        log_audit "deploy_failed" "version=${VERSION}, env=${ENV}"
        exit 1
    fi
}

# 执行主流程
main "$@"
```

### 3.2 健康检查脚本 health-check.sh

```bash
#!/bin/bash
######################################################################
# 健康检查脚本
######################################################################

set -euo pipefail

# 健康检查函数
check_health() {
    local url="${1}"

    info "检查 URL: ${url}"

    # 1. HTTP 状态码检查
    local http_code=$(curl -s -o /dev/null -w "%{http_code}" "${url}" --max-time 10 || echo "000")

    if [[ "${http_code}" != "200" ]]; then
        error "HTTP 状态码异常: ${http_code}"
        return 1
    fi

    # 2. 检查关键资源
    local content=$(curl -s "${url}" --max-time 10)

    # 检查关键 JS 文件
    if ! echo "${content}" | grep -q "main.js\|app.js\|index.js"; then
        error "关键 JS 文件不存在"
        return 1
    fi

    # 3. 检查关键关键字（可根据项目定制）
    if ! echo "${content}" | grep -q "<div id=\"app\"\|<div id=\"root\""; then
        error "应用根节点不存在"
        return 1
    fi

    # 4. 检查响应时间（可选）
    local response_time=$(curl -s -o /dev/null -w "%{time_total}" "${url}" --max-time 10)
    info "响应时间: ${response_time}s"

    if (( $(echo "${response_time} > 3" | bc -l) )); then
        warn "响应时间过长: ${response_time}s"
    fi

    return 0
}

# 导出函数供其他脚本使用
export -f check_health
```

### 3.3 回滚脚本 rollback.sh

```bash
#!/bin/bash
######################################################################
# 回滚脚本
######################################################################

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
source "${PROJECT_ROOT}/configs/deploy.conf"
source "${SCRIPT_DIR}/lib/logger.sh"
source "${SCRIPT_DIR}/lib/version.sh"
source "${SCRIPT_DIR}/notify.sh"

# 回滚到上一个版本
rollback() {
    local env="${1}"

    info "开始回滚 ${env} 环境..."

    # 获取环境配置
    case "${env}" in
        test)
            DEPLOY_BASE_DIR="${TEST_DEPLOY_BASE_DIR}"
            ;;
        production)
            DEPLOY_BASE_DIR="${PROD_DEPLOY_BASE_DIR}"
            ;;
        *)
            error "未知的环境: ${env}"
            return 1
            ;;
    esac

    local versions_file="${DEPLOY_BASE_DIR}/versions.json"
    local current_link="${DEPLOY_BASE_DIR}/current"
    local previous_link="${DEPLOY_BASE_DIR}/previous"

    # 读取当前和上一个版本
    if [[ ! -f "${versions_file}" ]]; then
        error "版本文件不存在: ${versions_file}"
        return 1
    fi

    local current_version=$(get_current_version)
    local previous_version=$(get_previous_version)

    if [[ -z "${previous_version}" ]]; then
        error "没有可回滚的版本"
        return 1
    fi

    info "当前版本: ${current_version}"
    info "回滚到版本: ${previous_version}"

    # 获取上一个版本的元数据
    local previous_dir="${DEPLOY_BASE_DIR}/releases/${previous_version}"
    if [[ ! -d "${previous_dir}" ]]; then
        error "上一个版本目录不存在: ${previous_dir}"
        return 1
    fi

    # 执行回滚
    local rollback_version="rollback-$(date +%Y%m%d-%H%M%S)"
    local current_target=$(readlink -f "${current_link}")

    # 保存当前版本为回滚点
    if [[ -n "${current_target}" ]]; then
        ln -sfn "${current_target}" "${DEPLOY_BASE_DIR}/rollback-point"
    fi

    # 切换到上一个版本
    ln -sfn "${previous_dir}" "${current_link}"

    # 更新 previous
    if [[ -L "${previous_link}" ]]; then
        local new_previous=$(readlink "${previous_link}")
        # 保持 previous 指向原来的 current（现在变成 previous）
    fi

    # 重载 Nginx
    sudo nginx -s reload

    info "回滚完成"

    # 记录回滚
    local versions_file="${DEPLOY_BASE_DIR}/versions.json"
    if command -v jq &> /dev/null; then
        local current_json=$(cat "${versions_file}")
        echo "${current_json}" | jq \
            --arg current "${previous_version}" \
            --arg previous "${current_version}" \
            '.current = $current | .previous = $previous | .history[0].status = "rolled_back"' \
            > "${versions_file}.tmp"
        mv "${versions_file}.tmp" "${versions_file}"
    fi

    return 0
}

# 导出函数
export -f rollback
```

### 3.4 通知脚本 notify.sh

```bash
#!/bin/bash
######################################################################
# 飞书通知脚本
######################################################################

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
source "${PROJECT_ROOT}/configs/deploy.conf"

# 发送飞书通知
send_feishu_notification() {
    local message="${1}"
    local at_all="${2:-false}"

    local webhook_url="${FEISHU_WEBHOOK_URL}"

    # 构建 @ 列表
    local at_users='[]'
    if [[ "${at_all}" == "true" ]]; then
        at_users='[{"open_id":"*"}]'
    fi

    # 发送消息
    curl -s -X POST "${webhook_url}" \
        -H 'Content-Type: application/json' \
        -d "{
            \"msg_type\": \"interactive\",
            \"card\": {
                \"header\": {
                    \"title\": {
                        \"tag\": \"plain_text\",
                        \"content\": \"CICD 部署通知\"
                    },
                    \"template\": \"${at_all}"
                },
                \"elements\": [{
                    \"tag\": \"div\",
                    \"text\": {
                        \"tag\": \"lark_md\",
                        \"content\": \"${message}\"
                    }
                }, {
                    \"tag\": \"action\",
                    \"actions\": [{
                        \"tag\": \"button\",
                        \"text\": {
                            \"tag\": \"plain_text\",
                            \"content\": \"查看 Pipeline\"
                        },
                        \"type\": \"default\",
                        \"url\": \"${GITLAB_URL}/-/pipelines/${PIPELINE_ID:-}\"
                    }, {
                        \"tag\": \"button\",
                        \"text\": {
                            \"tag\": \"plain_text\",
                            \"content\": \"访问应用\"
                        },
                        \"type\": \"primary\",
                        \"url\": \"${DEPLOY_URL}\"
                    }]
                }]
            }
        }" > /dev/null

    info "飞书通知已发送"
}

# 导出函数
export -f send_feishu_notification
```

### 3.5 公共库函数 lib/version.sh

```bash
#!/bin/bash
######################################################################
# 版本管理函数
######################################################################

# 获取当前版本
get_current_version() {
    local versions_file="${DEPLOY_BASE_DIR}/versions.json"

    if [[ -f "${versions_file}" ]]; then
        if command -v jq &> /dev/null; then
            jq -r '.current' "${versions_file}" 2>/dev/null
        else
            grep -oP '"current"\s*:\s*"\K[^"]+' "${versions_file}" || echo ""
        fi
    fi
}

# 获取上一个版本
get_previous_version() {
    local versions_file="${DEPLOY_BASE_DIR}/versions.json"

    if [[ -f "${versions_file}" ]]; then
        if command -v jq &> /dev/null; then
            jq -r '.previous' "${versions_file}" 2>/dev/null
        else
            grep -oP '"previous"\s*:\s*"\K[^"]+' "${versions_file}" || echo ""
        fi
    fi
}

# 导出函数
export -f get_current_version
export -f get_previous_version
```

### 3.6 公共库函数 lib/logger.sh

```bash
#!/bin/bash
######################################################################
# 日志函数
######################################################################

# 日志级别
readonly LOG_DEBUG=0
readonly LOG_INFO=1
readonly LOG_WARN=2
readonly LOG_ERROR=3

# 当前日志级别（可通过环境变量配置）
: "${LOG_LEVEL:=${LOG_INFO}}"

# 颜色定义
readonly COLOR_RESET='\033[0m'
readonly COLOR_DEBUG='\033[0;36m'    # Cyan
readonly COLOR_INFO='\033[0;32m'     # Green
readonly COLOR_WARN='\033[0;33m'     # Yellow
readonly COLOR_ERROR='\033[0;31m'    # Red

# 日志函数
log() {
    local level=$1
    shift
    local message="$*"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    if [[ ${level} -ge ${LOG_LEVEL} ]]; then
        case ${level} in
            ${LOG_DEBUG})
                echo -e "${COLOR_DEBUG}[DEBUG]${COLOR_RESET} ${timestamp} - ${message}"
                ;;
            ${LOG_INFO})
                echo -e "${COLOR_INFO}[INFO]${COLOR_RESET} ${timestamp} - ${message}"
                ;;
            ${LOG_WARN})
                echo -e "${COLOR_WARN}[WARN]${COLOR_RESET} ${timestamp} - ${message}"
                ;;
            ${LOG_ERROR})
                echo -e "${COLOR_ERROR}[ERROR]${COLOR_RESET} ${timestamp} - ${message}"
                ;;
        esac
    fi
}

debug() { log ${LOG_DEBUG} "$@"; }
info() { log ${LOG_INFO} "$@"; }
warn() { log ${LOG_WARN} "$@"; }
error() { log ${LOG_ERROR} "$@"; }

# 导出函数
export -f debug info warn error
```

---

## 四、配置文件

### 4.1 部署配置 deploy.conf

```bash
######################################################################
# 部署配置文件
######################################################################

# Git 配置
GIT_REPO_URL="git@your-gitlab.com:frontend/app.git"
GITLAB_URL="https://your-gitlab.com"

# 测试环境配置
TEST_PROJECT_NAME="frontend-app-test"
TEST_DEPLOY_BASE_DIR="/var/www/frontend-test"
TEST_DEPLOY_URL="http://test.example.com"
TEST_HEALTH_CHECK_URL="http://localhost:8080"
TEST_KEEP_RELEASES=3

# 生产环境配置
PROD_PROJECT_NAME="frontend-app-prod"
PROD_DEPLOY_BASE_DIR="/var/www/frontend"
PROD_DEPLOY_URL="https://www.example.com"
PROD_HEALTH_CHECK_URL="https://www.example.com"
PROD_KEEP_RELEASES=10

# Nginx 配置
NGINX_USER="nginx"
NGINX_CONFIG_DIR="/etc/nginx/conf.d"

# 健康检查配置
HEALTH_CHECK_MAX_RETRIES=30
HEALTH_CHECK_INTERVAL=2

# 飞书配置
FEISHU_WEBHOOK_URL="https://open.feishu.cn/open-apis/bot/v2/hook/xxx"

# OSS 配置
OSS_BUCKET="frontend-releases"
OSS_ENDPOINT="oss-cn-hangzhou.aliyuncs.com"
OSS_PATH="/releases/"
```

### 4.2 Nginx 配置模板

```nginx
# /etc/nginx/conf.d/frontend.conf

# 测试环境
server {
    listen 8080;
    server_name test.example.com;

    root /var/www/frontend-test/current;
    index index.html;

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_min_length 1000;

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # HTML 文件不缓存
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }

    # SPA 路由支持
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
    }
}

# 生产环境
server {
    listen 80;
    server_name www.example.com example.com;

    # 强制 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name www.example.com;

    # SSL 证书
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    root /var/www/frontend/current;
    index index.html;

    # 安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_min_length 1000;

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # HTML 文件不缓存
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }

    # SPA 路由支持
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
    }
}
```

---

## 五、部署流程说明

### 5.1 测试环境部署流程

```
┌────────────────────────────────────────────────────────────────┐
│                     测试环境部署流程                             │
└────────────────────────────────────────────────────────────────┘

1. 触发部署
   ├─ 开发人员在 GitLab 项目页面
   ├─ 进入 CI/CD > Pipelines
   └─ 点击 deploy:test 任务的 "Play" 按钮

2. GitLab CI/CD 执行
   ├─ 运行 build:frontend 任务
   ├─ 构建完成后，触发 deploy:test 任务
   └─ 通过 SSH 连接到部署服务器执行 deploy.sh

3. 部署服务器执行
   ├─ [1/10] 初始化部署环境
   ├─ [2/10] 拉取指定分支代码
   ├─ [3/10] 安装依赖并构建
   ├─ [4/10] 生成版本元数据
   ├─ [5/10] 更新版本索引文件
   ├─ [6/10] 部署应用（更新软链接）
   ├─ [7/10] 上传到 OSS（异步）
   ├─ [8/10] 重载 Nginx
   ├─ [9/10] 健康检查（30秒超时）
   └─ [10/10] 清理旧版本

4. 结果通知
   ├─ 成功：飞书通知包含访问链接
   └─ 失败：自动回滚 + 飞书告警

5. 人工验证
   ├─ 开发人员点击链接访问
   ├─ 验证功能正常
   └─ 结束 或 触发手动回滚
```

### 5.2 生产环境部署流程

```
┌────────────────────────────────────────────────────────────────┐
│                    生产环境部署流程                              │
└────────────────────────────────────────────────────────────────┘

1. 触发部署
   ├─ 开发人员在 GitLab 项目仓库
   ├─ 执行: git tag v1.2.3 && git push origin v1.2.3
   └─ GitLab CI/CD 自动检测到 Tag Push 事件

2. GitLab CI/CD 执行
   ├─ 验证 Tag 格式（/^v\d+\.\d+\.\d+$/）
   ├─ 运行 build:frontend 任务
   ├─ 构建完成后，触发 deploy:production 任务
   └─ 通过 SSH 连接到部署服务器执行 deploy.sh

3. 部署服务器执行
   ├─ [1/11] 初始化部署环境
   ├─ [2/11] 从 Tag 拉取代码
   ├─ [3/11] 安装依赖并构建
   ├─ [4/11] 生成版本元数据
   ├─ [5/11] 更新版本索引文件
   ├─ [6/11] 部署应用（更新软链接）
   ├─ [7/11] 上传到 OSS（异步）
   ├─ [8/11] 重载 Nginx
   ├─ [9/11] 健康检查（60秒超时）
   ├─ [10/11] 更新 .deploy-history 文件
   └─ [11/11] 清理旧版本（保留10个）

4. 结果通知
   ├─ 成功：飞书通知包含验证链接
   └─ 失败：自动回滚 + 飞书告警

5. 人工验证
   ├─ 负责人点击链接访问
   ├─ 验证功能正常
   ├─ 访问 GitLab MR（如需记录）
   └─ 结束 或 触发手动回滚
```

---

## 六、回滚方案

### 6.1 自动回滚触发条件

| 条件 | 说明 |
|------|------|
| 健康检查失败 | HTTP 状态码非 200 或关键资源缺失 |
| 部署超时 | 构建或部署超过预设时间（5分钟） |
| Nginx 配置错误 | nginx -t 测试失败 |

### 6.2 手动回滚操作

```bash
# 方式 1: 通过部署脚本
ssh deploy@server "/opt/frontend-deploy/scripts/deploy.sh --env=production --rollback"

# 方式 2: 在 GitLab CI/CD 中手动触发
# 配置 .gitlab-ci.yml 添加回滚任务
rollback:production:
  stage: deploy-prod
  script:
    - ssh deploy@server "/opt/frontend-deploy/scripts/deploy.sh --env=production --rollback"
  when: manual
  only:
    - /^v\d+\.\d+\.\d+$/
```

### 6.3 回滚流程

```
┌────────────────────────────────────────────────────────────────┐
│                      回滚流程                                    │
└────────────────────────────────────────────────────────────────┘

1. 保存当前状态
   └─ 将 current 指向的目录保存为 rollback-point

2. 切换版本
   ├─ current -> previous（上一个版本）
   └─ previous -> rollback-point（原当前版本）

3. 重载服务
   └─ nginx -s reload

4. 更新版本记录
   └─ versions.json 标记当前版本状态为 "rolled_back"

5. 发送通知
   └─ 飞书告警，包含回滚前后版本信息
```

---

## 七、监控和日志

### 7.1 日志文件位置

```
/opt/frontend-deploy/logs/
├── deploy.log        # 部署日志
├── error.log         # 错误日志
└── audit.log         # 审计日志（JSON 格式）
```

### 7.2 日志格式

```json
// audit.log 示例
{
  "timestamp": "2024-04-29T16:31:00Z",
  "event": "deploy_success",
  "details": "version=v1.2.3, env=production, triggerer=张三"
}
```

### 7.3 日志查看命令

```bash
# 查看部署日志
tail -f /opt/frontend-deploy/logs/deploy.log

# 查看错误日志
tail -f /opt/frontend-deploy/logs/error.log

# 查看审计日志
tail -f /opt/frontend-deploy/logs/audit.log | jq

# 统计部署成功率
cat /opt/frontend-deploy/logs/audit.log | \
  jq -r 'select(.event=="deploy_success" or .event=="deploy_failed") | .event' | \
  sort | uniq -c
```

---

## 八、安全建议

### 8.1 SSH 密钥管理

```bash
# 在部署服务器上生成专用 SSH 密钥
ssh-keygen -t ed25519 -C "deploy@frontend-app" -f ~/.ssh/deploy_key

# 将公钥添加到 GitLab 项目
# Settings > Repository > Deploy Keys
```

### 8.2 最小权限原则

```bash
# /etc/sudoers.d/deploy
deploy ALL=(ALL) NOPASSWD: /usr/sbin/nginx -t
deploy ALL=(ALL) NOPASSWD: /usr/sbin/nginx -s reload
```

### 8.3 密钥管理

```bash
# 使用环境变量或密钥管理系统
export OSS_ACCESS_KEY_ID="$(vault kv get -field=access_key secret/frontend/oss)"
export OSS_ACCESS_KEY_SECRET="$(vault kv get -field=secret_key secret/frontend/oss)"
```

---

## 九、故障排查

### 9.1 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 健康检查失败 | 构建产物不完整 | 检查构建日志，确认 dist 目录内容 |
| Nginx 404 | 软链接错误 | 检查 current 软链接指向 |
| Nginx 502 | 后端服务异常 | 检查后端服务状态 |
| OSS 上传失败 | 网络或认证问题 | 检查 OSS 配置和网络连接 |

### 9.2 排查命令

```bash
# 检查当前版本
ls -la /var/www/frontend/current

# 检查软链接
readlink -f /var/www/frontend/current

# 检查 Nginx 配置
nginx -t

# 查看 Nginx 错误日志
tail -f /var/log/nginx/error.log

# 手动健康检查
curl -I http://localhost:8080
```

---

## 十、实施检查清单

### 10.1 准备阶段

- [ ] 准备部署服务器（安装 Node.js、Nginx）
- [ ] 配置 SSH 访问（GitLab 到部署服务器）
- [ ] 配置 Nginx 虚拟主机
- [ ] 创建部署目录结构
- [ ] 配置 OSS 存储

### 10.2 配置阶段

- [ ] 创建 GitLab CI/CD 配置文件
- [ ] 配置 GitLab CI/CD 变量
- [ ] 上传部署脚本到服务器
- [ ] 配置 deploy.conf
- [ ] 配置飞书机器人 Webhook

### 10.3 测试阶段

- [ ] 测试环境手动部署测试
- [ ] 健康检查测试
- [ ] 自动回滚测试
- [ ] 飞书通知测试
- [ ] 手动回滚测试

### 10.4 上线阶段

- [ ] 生产环境 Tag 部署测试
- [ ] 完整流程验证
- [ ] 文档完善
- [ ] 团队培训

---

## 附录

### A. 快速命令参考

```bash
# 部署到测试环境
./deploy.sh --env=test --branch=develop

# 部署到生产环境（需通过 GitLab Tag 触发）
./deploy.sh --env=production --tag=v1.2.3

# 回滚
./deploy.sh --env=production --rollback

# 查看当前版本
cat /var/www/frontend/versions.json | jq '.current, .previous'

# 手动重载 Nginx
sudo nginx -s reload

# 清理旧版本
./cleanup.sh --env=test
```

### B. GitLab CI/CD Pipeline 状态图

```
┌─────────────────────────────────────────────────────────────────┐
│                       Pipeline 流程图                             │
└─────────────────────────────────────────────────────────────────┘

代码提交/Tag
    │
    ▼
┌─────────────┐
│ build       │ ◄─── develop, main, release/*, tags
└──────┬──────┘
       │
       ├──────────────────┐
       │                  │
       ▼                  ▼
┌──────────────┐   ┌──────────────┐
│ deploy:test  │   │ deploy:prod  │
│ (手动触发)    │   │ (Tag 自动)    │
└──────┬───────┘   └──────┬───────┘
       │                  │
       ▼                  ▼
   成功/回滚          成功/回滚
       │                  │
       └──────────────────┘
                │
                ▼
           飞书通知
```

---

*文档版本: v2.0*
*更新时间: 2024-04-29*
*适用环境: GitLab 私有化部署*
