# Pages

## 安装和注册gitlabrunner
```
curl -s https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
yum install -y gitlab-runner.x86_64-11.9.2-1
```

注册runner

<img  style="border:1px solid #F3F4F4" src="imgs/24.jpg"/>



TODO: gitlab与gitlab-runner版本额对应关系


## 启用Pages

```
vim /etc/gitlab/gitlab.rb
```
<img  style="border:1px solid #F3F4F4" src="imgs/25.jpg"/>

```
gitlab-ctl reconfigure
gitlab-ctl restart
```

## 第一个Pages站点