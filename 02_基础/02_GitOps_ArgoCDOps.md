
https://icloudnative.io/posts/what-is-gitops/


# 1 Gitops的四个原则

[GitOps](https://www.weave.works/technologies/gitops/) 这个概念最早是由 [Weaveworks](https://www.weave.works) 的 CEO Alexis Richardson 在 2017 年提出的，它是一种全新的基于 Git 仓库来管理 Kubernetes 集群和交付应用程序的方式。它包含以下四个基本原则：

1. **声明式（Declarative）**：整个系统必须通过声明式的方式进行描述，比如 Kubernetes 就是声明式的，它通过 YAML 来描述系统的期望状态；
2. **版本控制和不可变（Versioned and immutable）**：所有的声明式描述都存储在 Git 仓库中，通过 Git 我们可以对系统的状态进行版本控制，记录了整个系统的修改历史，可以方便地回滚；
3. **自动拉取（Pulled automatically）**：我们通过提交代码的形式将系统的期望状态提交到 Git 仓库，系统从 Git 仓库自动拉取并做出变更，这种被称为 Pull 模式，整个过程不需要安装额外的工具，也不需要配置 Kubernetes 的认证授权；而传统的 CI/CD 工具如 Jenkins 或 CircleCI 等使用的是 Push 模式，这种模式一般都会在 CI 流水线运行完成后通过执行命令将应用部署到系统中，这不仅需要安装额外工具（比如 kubectl），还需要配置 Kubernetes 的授权，而且这种方式无法感知部署状态，所以也就无法保证集群状态的一致性了；
4. **持续调谐（Continuously reconciled）**：通过在目标系统中安装一个 Agent，一般使用 Kubernetes Operator 来实现，它会定期检测实际状态与期望状态是否一致，一旦检测到不一致，Agent 就会自动进行修复，确保系统达到期望状态，这个过程就是调谐（Reconciliation）；这样做的好处是将 Git 仓库作为单一事实来源，即使集群由于误操作被修改，Agent 也会通过持续调谐自动恢复。

其实，在提出 GitOps 概念之前，已经有另一个概念 IaC （Infrastructure as Code，基础设施即代码）被提出了，IaC 表示使用代码来定义基础设施，方便编辑和分发系统配置，它作为 DevOps 的最佳实践之一得到了社区的广泛关注。关于 IaC 和 GitOps 的区别，可以参考 [The GitOps FAQ](https://www.weave.works/technologies/gitops-frequently-asked-questions/)。

# 2 GitOps工作流

![在这里插入图片描述](https://img-blog.csdnimg.cn/7cc28a8d3488488e83326f2913eb5e46.png)



我们可以把这个完整的 GitOps 工作流分成三个部分来看。
第一部分：开发者推送代码到 GitHub 仓库，然后触发 GitHub Action 自动构建。

第二部分：GitHub Action 自动构建，它包括下面三个步骤：
    构建示例应用的镜像。
    将示例应用的镜像推送到 Docker Registry 镜像仓库。
    更新代码仓库中 Helm Chart values.yaml 文件的镜像版本。

第三部分：核心是 ArgoCD，它包括下面两个步骤:
    通过定期 Poll 的方式持续拉取 Git 仓库，并判断是否有新的 commit。
    从 Git 仓库获取 Kubernetes 对象，与集群对象进行实时比较，自动更新集群内有差异的资源。

前边的环境已经准备好，现在准备创建 GitOps 工作流中的第三部分，也就是创建 ArgoCD 应用，实现 Kubernetes 资源的自动同步。


[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423174902155-175248226.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423174902155-175248226.png)


1：当开发人员将开发完成的代码推送到git仓库会触发CI制作镜像并推送到镜像仓库
2：CI处理完成后，可以手动或者自动修改应用配置，再将其推送到git仓库
3：GitOps会同时对比目标状态和当前状态，如果两者不一致会触发CD将新的配置部署到集群中
其中，目标状态是Git中的状态，现有状态是集群的里的应用状态。

---------


![](image/Pasted%20image%2020240710173904.png)

- 首先，团队中的任何一个成员都可以 Fork 仓库对配置进行更改，然后提交 Pull Request。
- 接下来会运行 CI 流水线，一般会做这么几件事情：验证配置文件、执行自动化测试、检测代码的复杂性、构建 OCI 镜像、将镜像推送到镜像仓库等等。
- CI 流水线运行完成后，团队中拥有合并代码权限的人将会将这个 Pull Request 合并到主分支中 。一般拥有这个权限的都是研发人员、安全专家或者高级运维工程师。
- 最后会运行 CD 流水线，将变更应用到目标系统中（比如 Kubernetes 集群或者 AWS） 。
整个过程完全自动化且透明，通过多人协作和自动化测试来保证了基础设施声明配置的健壮性。而传统的模式是其中一个工程师在自己的电脑上操作这一切，其他人不知道发生了什么，也无法对其操作进行 Review

# 3 为什么我们要做GitOps？
我们可以使用kubectl、helm等工具直接发布配置，但是这样会出现一个严重的问题，那就是密钥共享为了让CI系统能够自动的部署应用，我们需要将集群的访问密钥共享给它，这会带来潜在的安全问题。


# 4 使用Git存储库存储所需应用程序的配置
Argo CD遵循GitOps模式，使用Git存储库存储所需应用程序的配置。
Kubernetes清单可以通过以下几种方式指定:
1：kustomize应用程序
2：helm图表
3：ksonnet应用程序
4：jsonnet文件
5：基于YAML/json配置
6：配置管理插件配置的任何自定义配置管理工具

Argo CD实现为kubernetes控制器，它持续监视运行中的应用程序，并将当前的活动状态与期望的目标状态进行比较(如Git repo中指定的那样)。如果已部署的应用程序的活动状态偏离了目标状态，则认为是OutOfSync。Argo CD报告和可视化这些差异，同时提供了方法，可以自动或手动将活动状态同步回所需的目标状态。在Git repo中对所需目标状态所做的任何修改都可以自动应用并反映到指定的目标环境中。



# 5 Push vs Pull

CD 流水线有两种模式：Push 和 Pull。

## 5.1 Push 模式

目前大多数 CI/CD 工具都使用基于 Push 的部署模式，例如 Jenkins、CircleCI 等。这种模式一般都会在 CI 流水线运行完成后执行一个命令（比如 kubectl）将应用部署到目标环境中

![](image/Pasted%20image%2020240710174044.png)


这种 CD 模式的缺陷很明显：
- 需要安装配置额外工具（比如 kubectl）；
- 需要 Kubernetes 对其进行授权；
- 需要云平台授权；
- 无法感知部署状态。也就无法感知期望状态与实际状态的偏差，需要借助额外的方案来保障一致性。

Kubernetes 集群或者云平台对 CI 系统的授权凭证在集群或云平台的信任域之外，不受集群或云平台的安全策略保护，因此 CI 系统很容易被当成非法攻击的载体

## 5.2 Pull 模式

Pull 模式会在目标环境中安装一个 Agent，例如在 Kubernetes 集群中就靠 Operator 来充当这个 Agent。Operator 会周期性地监控目标环境的实际状态，并与 Git 仓库中的期望状态进行比较，如果实际状态不符合期望状态，Operator 就会更新基础设施的实际状态以匹配期望状态

![](image/Pasted%20image%2020240710174114.png)


只有 Git 的变更可以作为期望状态的唯一来源，除此之外，任何人都不可以对集群进行任何更改，即使你修改了，也会被 Operator 还原为期望状态，这也就是传说中的不可变基础设施。

目前基于 Pull 模式的 CD 工具有 Argo CD， Flux CD 以及 ks-devops。



# 6 Pull 的部署模式 的优点


一般 GitOps 首选的都是基于 Pull 的部署模式，因为这种模式有很多不可替代的优势。

更强大的安全保障
上面已经提到了，使用 GitOps 不需要任何 Kubernetes 或者云平台的凭证来执行部署，Kubernetes 集群内的 Argo CD 或者 Flux CD 只需要访问 Git 仓库，并通过 Pull 模式来更新即可。
另一方面，Git 由用于跟踪和管理代码变更的强大密码学支持，拥有对变更进行签名以证明作者身份和来源的能力，这是保障集群安全的关键。


Git 作为事实的唯一真实来源
因为所有的应用包括基础设施的声明式配置都保存在 Git 中，并把 Git 作为应用系统的唯一事实来源，因此可以利用 Git 的强大功能操作所有东西，例如版本控制、历史记录、审计和回滚等等，无需使用 kubectl 这样的工具来操作。



提高生产力
Git 也是开发人员非常熟悉的工具，通过 Git 不断迭代，可以提高生产率，加快开发和部署速度，更快地推出新产品，同时提高系统的稳定性和可靠性。



更容易合规的审计
使用 GitOps 的基础设施可以像任何软件项目一样使用 Git 来管理，所以同样可以对其进行质量审计。当有人需要对基础设施进行更改时，会创建一个 Pull Request，等相关人员对其进行 Code Review 之后，更改才可以应用到系统中。



