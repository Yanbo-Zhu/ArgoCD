

# 1 项目Project 管理

## 1.1 新建项目

`setting` -> `projects`->`NEW PROJECT`

![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155402833-2093731604.png)

几个重要的信息，项目的基础信息，可以添加名称和描述，添加label用来过滤。

![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155413191-780924944.png)

`SOURCE REPOSITORIES` 部署应用的源信息，设置之后创建的app属于这个项目，那么模板的源信息只能从这个url获得。

`DESTINATIONS`应用部署到的目的集群，如果应用属于这个项目那么同步的目的集群只能是项目中配置的集群。

![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155429621-1273416785.png)





# 2 app管理

创建app的时需要注意app与projects中配置的资源信息对应，不然会直接报错。

## 2.1 CLI创建app

```bash
argocd app create pre-front-jian-butler-admin-ui --project pre-jian-butler --repo https://git.kailinesb.com/ops/argocd-helm-template.git --path jian-butler-admin-ui/cluspre-front-jian-butler-admin-ui ter-pre/helm --dest-namespace jian-butler --dest-server https://kubernetes.default.svc --helm-set app_name=front-jian-butler-admin-ui --helm-set type=nginx

pre-front-jian-butler-admin-ui    # 创建的app名称
--project                         # 属于哪个项目
--repo                            # 部署的源仓库地址
--path                            # 模板在仓库的路径
--dest-namespace                  # app部署的命名空间
--dest-server                     # app部署到的k8s cluster
--helm-set k1=v1                  # 设置helm的变量，多个值重复使用 --helm-set
--help                            # 查看帮助信息
```

## 2.2 图形界面创建

![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155446607-1125710928.png)

`Application Name` app的名称。

`Project Name` app属于哪个项目的。

`SYNC POLICY` 同步的策略，可以选择手动或者自动。

![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155503908-508899808.png)

`Repository URL` git仓库的地址

`Path` 模板所在的路径

![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155515525-626512447.png)

![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155524353-1558271890.png)

如果路径正取会显示helm被渲染的所有变量。如果需要手动设置的，在设置之后会有一个锤子状的图案。  
![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155555951-586615524.png)

![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155603265-1784101246.png)





