


---



> [Reviewbot](https://github.com) 是七牛云开源的一个项目，旨在提供一个自托管的代码审查服务, 方便做 code review/静态检查, 以及自定义工程规范的落地。




---


静态检查不是个新鲜事。


我记得早在几年前，我们就调研并使用过 sonarqube 做静态检查，但当时并没有大范围的推广。主要原因在于，一是发现的问题多数是风格问题，较少能发现缺陷; 二是 sonarqube 社区版的 worker 数有限制，满足不了我们大规模代码扫描的需求。当然，也是因为前一个问题，感觉付费并不是很划算。


而由于七牛主要使用 golang 语言，所以在静态检查方面，我们基本使用 go vet/fmt/lint 等，再就是后来的 golangci\-lint，好像也够用了。


但随着代码仓库的增多，以及对工程规范的不断强化，我们越来越发现当前的落地方式，已经开始无法满足我们的需求。


### Linter 工具的引入与更新问题


以 golangci\-lint 为例，它是 go 语言 linters 聚合器，内置了 100\+ linters，使用也很简单, `golangci-lint run` 一条命令即可。但你知道吗？如果没有特殊配置，你这条命令其实仅仅执行其中的 6 个 linter，绝大部分 linters 都没有执行！


另外，工具本身需要更新，且很多时候我们也会自己定制 linter 工具，这种时候该怎么做呢？如果仅有少量仓库，可能还好，但如果仓库很多，那维护成本就上去了。


还有就是新业务，新仓库，如何保证相关的检查能够及时配置，相关的规范能够正确落地？


靠自觉一定是不行的。


### Linter 问题的发现与修复情况


如何确保发现的问题能够被及时修复？如何让问题能更及时、更容易的被修复？


埋藏在大量 raw log 中的问题，一定是不容易被发现的，查找起来很麻烦，体验很差。


历史代码仓库的存量问题，谁来改？改动就需要时间，但实际上很多业务研发可能并没有动力来跟进。同样，变动总是有风险的，有些 lint 问题修复起来也并不简单，如果因修复 lint 问题而引入风险，那就得不偿失了。


如果想了解当前组织内 lint 问题的分布及修复情况，又该怎么办呢？


### 如何解决，方向在哪里？


不可否认，linter 问题也是问题，如果每行代码都能进行充分的 lint 检查，那一定比不检查要强。


另一方面，组织内制定好的工程规范，落地在日常的开发流程中，那一定是希望被遵守的，这类就是强需。


所以这个事情值得做，但做的方式是值得思考的，尤其是当我们有更高追求时。


参考 `CodeCov` 的服务方式，以及 `golangci-lint` `reviewdog` 等工具的设计理念，我们认为:


* 如果能对于新增仓库、历史仓库，不需要专人配置 job，就能自动生效，那一定是优雅的
* 如果能只针对 PR/MR 中的变动做分析和反馈，类似我们做 Code Review 那样，那对于提 PR 的同学来讲一定是优雅的，可接受的，随手修复的可能性极大
	+ 而进一步，针对 PR/MR 中涉及的文件中的历史代码进行反馈，在合理推动下，支持夹带修改，持续改进的可能性也会大大增强
* Lint 工具多种多样，或者我们自己开发出新工具时，能够较为轻松的让所有仓库都自动生效，那也一定是非常赞的，不然就可能陷入工具越多负担越重的风险


基于上面的思考，我认为我们需要的是: **一个中心化的 Code Review/静态检查服务，它能自动接受整个组织内 PR/MR 事件，然后执行各种预定义的检查，并给与精确到变动文 �� 级的有效反馈。它要能作为代码门禁，持续的保障入库代码质量。**


而 [Reviewbot](https://github.com):[veee加速器](https://youhaochi.com) 就是这样一个项目。


### Reviewbot 在设计和实现上有哪些特点？


**面向改进的反馈方式**


这将是 Reviewbot 反馈问题的核心方式，它会尽可能充分利用各 Git 平台的自身能力，精确到变动的代码行，提供最佳的反馈体验。


* Github Check Run (Annotations)
![](https://img2024.cnblogs.com/blog/293394/202411/293394-20241107144723141-774207240.png)
* Github Pull Request Review (Comments)
![](https://img2024.cnblogs.com/blog/293394/202411/293394-20241107144732582-1426411028.png)


**支持多种 Runner**


Reviewbot 是自托管的服务，推荐大家在企业内自行部署，这样对私有代码更友好。


Reviewbot 自身更像个管理服务，不限制部署方式。而对于任务的执行，它支持多种 Runner，以满足不同的需求。比如:


* 不同的仓库和 linter 工具，可能需要不同的基础环境，这时候你就可以将相关的环境做成 docker 镜像，直接通过 docker 来执行
* 而当任务较多时，为了执行效率，也可以选择通过 kubernetes 集群来执行任务。


使用也很简单，在配置文件中的目标仓库指定即可。类似:



```
dockerAsRunner:
  image: "aslan-spock-register.qiniu.io/reviewbot/base:go1.22.3-gocilint.1.59.1"

```


```
kubernetesAsRunner:
  image: "aslan-spock-register.qiniu.io/reviewbot/base:go1.23.2-gocilint.1.61.0"
  namespace: "reviewbot"

```

**零配置\+定制化**


本质上，Reviewbot 也是个 webhook 服务，所以我们只需要在 git provider 平台配置好 Reviewbot 的回调地址即可(github 也可以是 Github App)。


绝大部分的 linter 的默认最佳执行姿势都已经固定到代码中，如无特殊，不需要额外配置就能对所有仓库生效。


而如果仓库需要特殊对待，那就可以通过配置来调整。


类似:



```
org/repo:
  linters:
    golangci-lint:
      enable: true
      dockerAsRunner:
        image: "aslan-spock-register.qiniu.io/reviewbot/base:go1.22.3-gocilint.1.59.1"
      command:
        - "/bin/sh"
        - "-c"
        - "--"
      args:
        - |
          source env.sh
          export GO111MODULE=auto
          go mod tidy
          golangci-lint run --timeout=10m0s --allow-parallel-runners=true --print-issued-lines=false --out-format=line-number >> $ARTIFACT/lint.log 2>&1

```

**可观察**


Reviewbot 是在对工程规范强管理的背景下产生的，那作为工程规范的推动方，我们自然有需求想了解组织内当前规范的执行情况。比如, 都有哪些问题被检出？哪些仓库的问题最多？哪些仓库需要特殊配置？


目前 Reviewbot 支持通过企业微信来接收通知，比如:


* 检出有效问题


![](https://img2024.cnblogs.com/blog/293394/202411/293394-20241107144751642-1786813299.png)


* 遇到错误


![](https://img2024.cnblogs.com/blog/293394/202411/293394-20241107144804684-512843466.png)


当然，未来可能也会支持更多方式。



> 其他更多的功能和姿势，请参考仓库: [https://github.com/qiniu/reviewbot](https://github.com)


### Reviewbot 的未来规划


作为开源项目，Reviewbot 还需要解决很多可用性和易用性问题，以提升用户体验，比如最典型的，接入更多的 git provider(gitlab/gitee 等)，支持 CLI 模式运行。


但我个人认为，作为 code review 服务，提供更多的检测能力，才是重中之重。因为这不光是行业需求，也是我们自身需要。


所以后面我们除了会引入七牛内部推荐的规范，也会调研和探索更多的行业工具，同时会考虑引入 AI，探索 AI 在 code review 中的应用等等。


Anyway，Reviewbot 还很年轻，我们在持续的改进中，非常欢迎大家试用并提出宝贵意见。当然，更欢迎大家一起参与到项目建设中来。


感谢大家。


