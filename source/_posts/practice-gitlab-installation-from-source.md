title: 源码安装GitLab（MySQL & HTTPS）
date: 2018-01-12 18:26:13
tags: 
    - gitlab
    - mysql
    - nginx
    - practice
categories:
    - 实践
---

介绍一下在Ubuntu16.04系统下源码安装GitLab10.04且以MySQL作为数据库、SSL加密传输的步骤：

<!--more-->

## 开始

### Dependencies
安装依赖：
```cmd
$ sudo apt-get install -y build-essential zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libre2-dev libreadline-dev libncurses5-dev libffi-dev curl openssh-server checkinstall libxml2-dev libxslt-dev libcurl4-openssl-dev libicu-dev logrotate rsync python-docutils pkg-config cmake
```

为GitLab创建一个系统用户git：
```cmd
$ sudo adduser --disabled-login --gecos 'GitLab' git
```

$ sudo apt-get install -y postfix

### Git
安装最新版Git：
```cmd
$ sudo add-apt-repository ppa:git-core/ppa
$ sudo apt-get update
$ sudo apt-get install git
```


### Ruby
安装最新版Ruby：
```cmd
$ sudo apt-add-repository ppa:brightbox/ruby-ng
$ sudo apt-get update
$ sudo apt-get install ruby
$ sudo apt-get install ruby-dev
```

安装Bundler：
```cmd
$ sudo gem install bundler --no-ri --no-rdoc
```

### Go
安装Go1.9.2：
```cmd
$ curl --remote-name --progress https://dl.google.com/go/go1.9.2.linux-amd64.tar.gz
$ sudo tar -C /usr/local -xzf go1.9.2.linux-amd64.tar.gz
$ sudo ln -sf /usr/local/go/bin/{go,godoc,gofmt} /usr/local/bin/
$ rm go1.9.2.linux-amd64.tar.gz
```

### Node
安装NodeJS8.x：
```cmd
$ curl --location https://deb.nodesource.com/setup_8.x | sudo bash -
$ sudo apt-get install -y nodejs
```
安装yarn：
```cmd
$ curl --silent --show-error https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
$ sudo apt-get update
$ sudo apt-get install yarn
```

### MySQL
安装MySQL5.7，先从[MySQL官网](https://dev.mysql.com/downloads/repo/apt/)下载APT配置器(mysql-apt-config_0.8.9-1_all.deb)[https://dev.mysql.com/downloads/file/?id=474129]，然后：
```cmd
$ sudo dpkg -i mysql-apt-config_0.8.9-1_all.deb
$ sudo apt-get update
$ sudo apt-get install -y mysql-server mysql-client libmysqlclient-dev
```

MySQL安全配置：
```cmd
$ 
$ mysql --version
$ sudo mysql_secure_installation
```

创建GitLab用户和数据库：
```
$ mysql -u root -p
mysql> CREATE USER 'git'@'localhost' IDENTIFIED BY '$password';

# Create the GitLab production database
mysql> CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_general_ci`;

# Grant the GitLab user necessary permissions on the database
mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, CREATE TEMPORARY TABLES, DROP, INDEX, ALTER, LOCK TABLES, REFERENCES, TRIGGER ON `gitlabhq_production`.* TO 'git'@'localhost';

# Quit the database session
mysql> \q
```

测试用户是否可以访问数据库：
```cmd
$ sudo -u git -H mysql -u git -p -D gitlabhq_production
```

### Redis
安装Redis3.0.6：
```cmd
$ sudo apt-get install redis
```

### GitLab
#### 克隆GitLab源码
克隆GitLab源码，并配置GitLab（修改域名等等）：
```cmd
$ cd /home/git
$ sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b 10-4-stable gitlab
$ cd /home/git/gitlab
$ sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml
$ sudo -u git -H vi config/gitlab.yml
```

#### 配置GitLab
配置密钥文件：
```cmd
$ sudo -u git -H cp config/secrets.yml.example config/secrets.yml
$ sudo -u git -H chmod 0600 config/secrets.yml
```

配置Unicorn（修改监听端口等等）：
```cmd
$ sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb
$ sudo -u git -H vi config/unicorn.rb
```

设置目录权限：
```cmd
$ sudo chown -R git log/
$ sudo chown -R git tmp/
$ sudo chmod -R u+rwX,go-w log/
$ sudo chmod -R u+rwX tmp/

$ sudo chmod -R u+rwX tmp/pids/
$ sudo chmod -R u+rwX tmp/sockets/

$ sudo -u git -H mkdir public/uploads/
$ sudo chmod 0700 public/uploads

$ sudo chmod -R u+rwX builds/
$ sudo chmod -R u+rwX shared/artifacts/
$ sudo chmod -R ug+rwX shared/pages/
```

配置Rack Attack：
```cmd
$ sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb
```

配置Resque（修改Redis链接等等）：
```cmd
$ sudo -u git -H cp config/resque.yml.example config/resque.yml
$ sudo -u git -H vi config/resque.yml
```

配置Git用户：
```cmd
# Configure Git global settings for git user
# 'autocrlf' is needed for the web editor
$ sudo -u git -H git config --global core.autocrlf input

# Disable 'git gc --auto' because GitLab already runs 'git gc' when needed
$ sudo -u git -H git config --global gc.auto 0

# Enable packfile bitmaps
$ sudo -u git -H git config --global repack.writeBitmaps true

# Enable push options
$ sudo -u git -H git config --global receive.advertisePushOptions true
```

#### 数据库设置
配置MySQL（修改账户密码等等）：
```
$ sudo -u git cp config/database.yml.mysql config/database.yml
$ sudo -u git -H chmod o-rwx config/database.yml
$ sudo -u git -H vi config/database.yml
```

#### 安装Gems
安装Gems：
```cmd
# If you use MySQL (note, the option says "without ... postgres")
# If you want to use Kerberos for user authentication, then omit kerberos in the --without option
$ sudo -u git -H bundle install --deployment --without development test postgres aws kerberos
```

#### 安装GitLab Shell
安装GitLab Shell，并配置（修改GitLab链接等等）：
```cmd
# Run the installation task for gitlab-shell (replace `REDIS_URL` if needed):
$ sudo -u git -H bundle exec rake gitlab:shell:install RAILS_ENV=production SKIP_STORAGE_VALIDATION=true

# By default, the gitlab-shell config is generated from your main GitLab config.
# You can review (and modify) the gitlab-shell config as follows:
$ sudo -u git -H vi /home/git/gitlab-shell/config.yml
```

#### 安装Gitlab Workhorse
安装Gitlab Workhorse：
```cmd
$ sudo -u git -H bundle exec rake "gitlab:workhorse:install[/home/git/gitlab-workhorse]" RAILS_ENV=production

$ sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production GITLAB_ROOT_PASSWORD='$root_password' GITLAB_ROOT_EMAIL='$root_email'
```

#### secrets.yml
`/home/git/gitlab/config/secrets.yml`文件存储着各类数据加密的钥匙，应备份在一个安全的地方。

#### 安装启动脚本
安装在系统中启动GitLab服务的脚步：
```cmd
$ sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
$ sudo cp lib/support/init.d/gitlab.default.example /etc/default/gitlab
$ sudo update-rc.d gitlab defaults 21
```

#### 安装Gitaly
安装Gitaly，并配置：
```cmd
$ sudo -u git -H bundle exec rake "gitlab:gitaly:install[/home/git/gitaly]" RAILS_ENV=production
$ sudo chmod 0700 /home/git/gitlab/tmp/sockets/private
$ sudo chown git /home/git/gitlab/tmp/sockets/private
$ sudo -u git -H vi /home/git/gitaly/config.toml
```

#### 设置Logrotate
```cmd
$ sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab
```

#### 编译GetText PO 
```cmd
$ sudo -u git -H bundle exec rake gettext:compile RAILS_ENV=production
```

#### MySQL字符串限制
```cmd
$ sudo -u git -H bundle exec rake add_limits_mysql RAILS_ENV=production
```

#### 编译前端资源
```cmd
$ sudo -u git -H yarn install --production --pure-lockfile
$ sudo -u git -H bundle exec rake gitlab:assets:compile RAILS_ENV=production NODE_ENV=production
```

#### 启动GitLab服务
启动或重启GitLab服务：
```cmd
$ sudo service gitlab start
# or
$ sudo /etc/init.d/gitlab restart
```

#### 获取服务配置
```cmd
$ sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production
```

#### 检测服务状态
```cmd
$ sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production
```


### Nginx
安装最新版NginX：
```cmd
$ sudo add-apt-repository ppa:nginx/stable
$ sudo apt-get update
$ sudo apt-install nginx
```

配置NginX代理（修改域名等等）：
```cmd
$ sudo cp lib/support/nginx/gitlab /etc/nginx/sites-available/gitlab
$ sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab
$ sudo vi /etc/nginx/sites-available/gitlab
```

重启NginX服务：
```cmd
$ sudo service nginx start
```

### *Letsencrypt
安装Letsencrypt：
```cmd
$ sudo apt-get install letsencrypt
```

申请密钥和证书：
```cmd 
$ sudo letsencrypt sudo letsencrypt certonly --webroot -w /home/git/gitlab/public -d '$domainname'
```

配置NginX代理（修改域名、密钥位置、证书位置等等）：
```cmd
sudo cp lib/support/nginx/gitlab-ssl /etc/nginx/sites-available/gitlab
sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab
sudo vi /etc/nginx/sites-available/gitlab
```

重启NginX服务：
```cmd
$ sudo service nginx start
```

## 参考
* [Installation from source](https://docs.gitlab.com/ce/install/installation.html)
* [Database MySQL](https://docs.gitlab.com/ce/install/database_mysql.html)
