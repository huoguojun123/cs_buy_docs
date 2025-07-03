# CI/CD (持续集成/持续部署)

项目采用 **GitLab CI/CD** 作为自动化工具链的核心，实现了从代码提交到最终部署的全流程自动化，极大地提升了开发效率和交付质量。

## 1. CI/CD 核心流程

下图清晰地描绘了从开发者提交代码到服务上线所需的完整流程。

![CI/CD 流程图](./cicd_flow.puml)

**流程解读**:
1.  **开发与提交**: 开发者在本地完成功能开发，并将代码推送到一个`feature-`分支。
2.  **代码合并**: 开发者通过GitLab创建一个合并请求（Merge Request），请求将`feature-`分支合并到`develop`分支。此时会触发代码审查（Code Review）。
3.  **首次CI触发 (测试与构建)**: 合并请求被接受后，GitLab CI/CD被触发。CI Runner会执行预设的脚本，包括：
    -   运行所有后端服务的**单元测试** (`mvn test`)。
    -   执行**代码构建** (`mvn package` for backend, `npm run build` for frontend)。
4.  **自动部署到测试环境**: 一旦代码成功合并到`develop`分支，CI/CD会自动触发一个部署作业，将最新的服务版本部署到**测试环境 (Staging)**。
5.  **手动测试**: 开发或测试人员在测试环境对新功能进行验证。
6.  **发布到生产环境**:
    -   当功能验证通过，准备上线时，负责人会在`main`或`master`分支上创建一个Git **Tag** (例如 `v1.2.0`)。
    -   创建Tag会触发**生产环境部署**的Pipeline。
    -   该Pipeline是**手动触发**的 (`when: manual`)，需要有权限的人员在GitLab界面上点击确认，作为发布的最后一道防线。
    -   生产部署流程包括：构建Docker镜像、推送到私有镜像仓库、更新Kubernetes集群中的Deployment（触发滚动更新）。


## 2. GitLab CI 配置文件解析 (`.gitlab-ci.yml`)

CI/CD的所有行为都由项目根目录下的 `.gitlab-ci.yml` 文件定义。

```yaml
# 定义整个Pipeline的各个阶段，按顺序执行
stages:
  - build   # 构建阶段
  - test    # 测试阶段
  - deploy  # 部署阶段

# 定义全局变量，可在所有job中使用
variables:
  # 缓存Maven依赖，加快构建速度
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

# --- 后端CI/CD作业 ---

# 作业1：构建后端服务
build-backend:
  stage: build
  script:
    - mvn clean package -DskipTests # 打包所有Java微服务，跳过测试以加快构建
  artifacts:
    paths:
      - '**/target/*.jar' # 将所有生成的jar包作为产物，传递给后续阶段
    expire_in: 1 hour # 产物保留1小时

# 作业2：运行后端测试
test-backend:
  stage: test
  script:
    - mvn test # 独立运行单元测试

# 作业3：自动部署到测试环境
deploy-to-staging:
  stage: deploy
  script:
    - ./deploy.sh staging # 执行自定义的部署脚本
  rules:
    # 规则：当提交到develop分支时，执行此作业
    - if: '$CI_COMMIT_BRANCH == "develop"'

# 作业4：手动部署到生产环境
deploy-to-production:
  stage: deploy
  script:
    - ./build-docker-image.sh ${CI_COMMIT_TAG}
    - docker push registry.example.com/mall/${CI_PROJECT_NAME}:${CI_COMMIT_TAG}
    - ./update-k8s-deployment.sh ${CI_COMMIT_TAG}
  rules:
    # 规则1：当创建了tag时
    - if: '$CI_COMMIT_TAG'
      # 规则2：需要手动在GitLab UI中点击运行时才执行
      when: manual

# --- 前端CI/CD作业 ---

# 作业5：构建前端应用
build-frontend:
  stage: build
  image: node:16 # 使用指定的Docker镜像来运行作业
  script:
    - cd e-business-front
    - npm install
    - npm run build
  artifacts:
    paths:
      - e-business-front/dist/ # 保存构建产物

# 作业6：部署前端应用
deploy-frontend:
  stage: deploy
  script:
    # 将构建好的静态文件通过scp拷贝到Nginx服务器
    - scp -r e-business-front/dist/* user@frontend-server:/var/www/html/
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```
这个配置文件清晰地将前后端的CI/CD流程分离，并通过`rules`关键字精确控制了每个作业的触发条件，实现了`develop`分支自动部署、`tag`手动部署的典型Git Flow实践。 