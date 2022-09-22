---
layout: post
title: Sophos UTM利用分析——清除Last WebAdmin Sessions记录
---


## 0x00 前言
---

对于Sophos UTM设备，在web管理页面中，Last WebAdmin Sessions会记录用户每次登录的信息，本文仅在技术研究的角度介绍清除指定Last WebAdmin Sessions记录的方法，记录研究细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 研究过程
- 实现方法

## 0x02 Last WebAdmin Sessions简介
---

在web管理页面中，选中`Management`后会显示Last WebAdmin Sessions记录，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-6-6/2-1.png)

记录包括以下内容：

- User：登录用户名
- Start：登录时间
- State：退出时间
- IP address：登录IP
- Changelog：修改的配置

对于Changelog，点击`Show`，会显示修改的配置，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-6-6/2-2.png)

默认配置下，Last WebAdmin Sessions会显示最近的20条记录

## 0x03 研究过程
---

### 1.尝试修改`/var/confd/var/storage/cfg`

在上篇文章[《Sophos UTM利用分析——导出配置文件》](https://3gstudent.github.io/Sophos-UTM%E5%88%A9%E7%94%A8%E5%88%86%E6%9E%90-%E5%AF%BC%E5%87%BA%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)提到，`/var/confd/var/storage/cfg`存储Sophos UTM的配置信息，所以猜测通过修改`/var/confd/var/storage/cfg`文件可以实现Last WebAdmin Sessions记录的清除

`/var/confd/var/storage/cfg`的文件格式为Perl Storable files，这里使用[StorableEdit](https://viereck.ch/perl/storableedit/)来编辑文件

向Sophos UTM上传文件storableedit-1.5.pl，执行命令：

```
./storableedit-1.5.pl cfg
```

结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-6-6/3-1.png)

解析出的文件结构同使用[SophosUTM_ConfigParser.py](https://github.com/3gstudent/Homework-of-Python/blob/master/SophosUTM_ConfigParser.py)导出的结果一致

查看配置信息，命令如下：

```
cd lastchange
cd REF_AaaGroGroup1
ls
```

将所有属性置空，命令如下：

```
$cur->{'user'} = '',$cur->{'time'} = '',$cur->{'sid'} = '',$cur->{'srcip'} = ''
```

保存文件，命令如下：

```
x
```

然而，修改cfg文件后不会影响Last WebAdmin Sessions记录

### 2.反编译web管理页面的源码

web管理页面的程序文件路径：`/var/sec/chroot-httpd/var/webadmin/webadmin.plx`

使用[SophosUTM_plxDecrypter.py](https://github.com/3gstudent/Homework-of-Python/blob/master/SophosUTM_plxDecrypter.py)反编译`/var/sec/chroot-httpd/var/webadmin/webadmin.plx`

定位到关键文件：`export-webadmin.plx\wfe\asg\modules\asg_dashboard.pm`

定位到关键内容：`my $userlog = $sys->userlog_read(max => 20, facility => 'webadmin,acc-agent,acc_sso') || [];`

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-6-6/3-2.png)

从输出结果中定位到关键函数：`userlog_read`

### 3.定位关键函数`userlog_read`

google搜索`$sys->userlog_read`，找到一份参考文档：https://community.sophos.com/utm-firewall/astaroorg/f/asg-v8-000-beta-closed/69661/7-920-bug-open-failed-smtp-relay-login-is-showing-up-on-last-webadmin-logins

文档中有关于`userlog_read`的一些描述，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-6-6/3-3.png)

从描述得出，`userlog_read`同`cc`命令存在关联

### 4.反编译`cc`命令对应的进程

`cc`命令对应的文件为`/var/confd/confd.plx`，使用[SophosUTM_plxDecrypter.py](https://github.com/3gstudent/Homework-of-Python/blob/master/SophosUTM_plxDecrypter.py)反编译`/var/confd/confd.plx`

### 5.获得函数`userlog_read`细节

搜索`userlog_read`相关内容，命令如下：

```
grep -iR "userlog_read" /home/kali/1/decrypt/Export-confd.plx
```

输出结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-6-6/3-4.png)

从输出结果中定位关键文件：`Export-confd.plx/Info/webadmin/log.pm`

定位到函数定义：

```
sub userlog_read {
  my ($self, %args) = @_;
  $args{max} = $args{sid} ? 1 : $args{max} || 20;
  $args{facility} = { map {($_ => 1)} split /,/, $args{facility} }
    if $args{facility};

  my $sessions;
  $sessions = _consult_db($self, \%args)
    unless $self->get(qw(reporting userlog_from_logs));
  $sessions = _iterate_files($self, \%args)
    unless ref $sessions eq 'ARRAY';

  foreach my $sd (@$sessions) {
    $sd->{state} = (-e "$config::session_dir/$sd->{sid}" ? 'active' : 'ended')
      if ! $sd->{state} || $sd->{state} eq 'active';
  }

  return $sessions;
}
```

### 6.函数`userlog_read`代码分析

代码涉及两个操作，分别为读取数据库和读取文件，详情如下：

#### (1)数据库操作

关键代码：

```
sub _consult_db {
  my ($self, $args) = @_;

  my $facility_selection = '';
  $facility_selection = 'WHERE facility in ('.
    join( ',', map { '?' } keys %{$args->{facility}} ).') '
    if $args->{facility};
  my %sql = (
    sessions => 'SELECT sid, facility, srcip, username, time, endtime, state '
                .'FROM confd_sessions '.$facility_selection
                .'ORDER BY time DESC LIMIT ?',
    session  => 'SELECT sid, facility, srcip, username, time, endtime, state '
                .'FROM confd_sessions WHERE sid = ?',
    nodes    => 'SELECT * FROM confd_nodes WHERE sid = $1 ORDER BY time DESC',
    objects  => 'SELECT * FROM confd_objects WHERE sid = $1 ORDER BY time DESC',
  );

  # Prepare database access.
  my $db = Astaro::ADBS->new(dbName => 'reporting') or return;
  while (my ($key, $query) = each %sql) {
    $db->registerSQL($key, $query) or return;
  }

  # List Confd sessions.
  my $sessh;
  if ($args->{sid}) {
    $sessh = $db->getHandle('session') or return;
    $sessh->execute($args->{sid}) or return;
  } elsif( $args->{facility} ) {
    $sessh = $db->getHandle('sessions') or return;
    $sessh->execute(keys %{$args->{facility}}, $args->{max}) or return;
  } else {
    $sessh = $db->getHandle('sessions') or return;
    $sessh->execute($args->{max}) or return;
  }
  my $sessions = $sessh->fetchall_arrayref({});
  my $nodeh = $db->getHandle('nodes') or return;
  my $objh = $db->getHandle('objects') or return;
  foreach my $sd (@$sessions) {

    # Tweak session data.
    $sd->{time} =~ tr/- /:-/;
    $sd->{endtime} =~ tr/- /:-/ if defined $sd->{endtime};
    $sd->{user} = delete $sd->{username};  # user is a reserved word in SQL
    $sd->{user} .= ' (SUM)' if $sd->{facility} eq 'acc_sso';
    $sd->{user} = utils::Sanitize::sanitize($sd->{user}) if $sd->{user};

    # Fetch node changes.
    $nodeh->execute($sd->{sid}) or return;
    foreach my $node (@{ $nodeh->fetchall_arrayref({}) }) {
      $node->{time} =~ tr/- /:-/;
      $node->{node_descr} = Message::get_phrase(
        'N', $node, { Nattrs => ['node'] });
      $sd->{main}{$node->{node}} ||= [];
      push @{$sd->{main}{$node->{node}}}, $node;
    }
    
    # Fetch object changes.
    $objh->execute($sd->{sid}) or return;
    foreach my $object (@{ $objh->fetchall_arrayref({}) }) {
      my $attrs = $object->{attrs} || [];
      $object->{attributes} = [];
      while (@$attrs) {
        my $name = shift @$attrs;
        $object->{"attr_$name"} = shift @$attrs;
        $object->{"oldattr_$name"} = shift @$attrs;
        $object->{"descr_$name"} = Message::get_phrase(
          'A', $object, { attr => $name });
        push @{$object->{attributes}}, $name;
      }
      delete $object->{attrs};
      if (@{$object->{attributes}}) {
        $object->{attributes} = [ sort @{$object->{attributes}} ];
      } else {
        delete $object->{attributes};
      }
      $object->{time} =~ tr/- /:-/;
      $object->{obj_descr} = Message::get_phrase('O', $object, {});
      $sd->{objects}{$object->{ref}} ||= [];
      push @{$sd->{objects}{$object->{ref}}}, $object;
    }
  }
  $db->disconnect;

  return $sessions;
}
```

代码分析：

从数据库`reporting`中分别执行以下操作实现数据读取：

```
sessions：   SELECT sid,facility,srcip,username,time,endtime,state FROM confd_sessions;
nodes：      SELECT * FROM confd_nodes;
objects：    SELECT * FROM confd_objects;
```

经过测试分析，`confd_sessions`存储Session信息

读取Session信息的cmd命令：

```
psql reporting -U postgres -c 'SELECT sid,facility,srcip,username,time,endtime,state FROM confd_sessions;'
```

#### (2)文件操作

关键代码：

```
sub _iterate_files {
  my ($self, $args) = @_;

  # choose the first file to process
  my $filename = '/var/log/confd.log';
  if (defined $args->{time}) {
    my @then;
    if ($args->{time} =~ /^(\d{4}):(\d\d):(\d\d)/) {
      @then = (0, 0, 12, $3, $2-1, $1-1900);
    } else {
      @then  = localtime($args->{time});
    }
    my $then  = POSIX::strftime('%F', @then);
    my $now   = POSIX::strftime('%F', localtime);
    $filename = POSIX::strftime(
      '/var/log/confd/%Y/%m/confd-%Y-%m-%d.log.gz',
      @then,
    ) if $then ne $now;
  }

  # process the first file
  my $sessions = [];
  my $sdata = {};
  _parse_file($self, $filename, $sessions, $sdata, $args);

  # if needed, process archived log files
  if (@$sessions < $args->{max} && not $args->{time}) {
    my $iter = File::Next::files({
      file_filter => sub { /\.log\.gz$/ },
      sort_files => \&File::Next::sort_reverse,
    }, '/var/log/confd');
    while (@$sessions < $args->{max}) {
      $filename = $iter->();
      last unless defined $filename;
      my @new_sessions;
      _parse_file($self, $filename, \@new_sessions, $sdata, $args);
      push @$sessions, @new_sessions;
    }
  }

  # limit the number of sessions to report on
  splice @$sessions, $args->{max} if @$sessions >= $args->{max};
  return [ @{$sdata}{@$sessions} ];
}
```

代码分析：

读取文件`/var/log/confd.log`，`/var/log/confd.log`只能保存现在时间到之前一段时间的日志，更早时间的日志会保存在`/var/log/confd/%Y/%m/confd-%Y-%m-%d.log.gz`，例如2022年5月16日的日志对应位置为`/var/log/confd/2022/05/confd-2022-05-16.log.gz`

经过测试分析，`/var/log/confd.log`存储Session信息

### 7.编辑文件中存储的Session信息

查看登录成功的信息：

```
cat /var/log/confd.log| grep success
```

返回结果示例：

```
2022:05:23-00:19:33 test confd[41177]: I Role::authenticate:185() => id="3106" severity="info" sys="System" sub="confd" name="authentication successful" user="admin" srcip="192.168.1.2" sid="8ad7bbf2781b006d99176eea9050694811e745e04acfab3dd0179620109a41ab" facility="webadmin" client="webadmin.plx" call="new"<31>May 23 00:19:33 confd[41177]: D sys::AUTOLOAD:307() => id="3100" severity="debug" sys="System" sub="confd" name="external call" user="admin" srcip="192.168.1.2" facility="webadmin" client="webadmin.plx" lock="none" method="get_SID"
```

从结果中获得sid为`8ad7bbf2781b006d99176eea9050694811e745e04acfab3dd0179620109a41ab`

筛选出指定sid的信息：

```
cat /var/log/confd.log| grep 8ad7bbf2781b006d99176eea9050694811e745e04acfab3dd0179620109a41ab
```

返回结果示例：

```
2022:05:23-00:19:33 test confd[41177]: I Role::authenticate:185() => id="3106" severity="info" sys="System" sub="confd" name="authentication successful" user="admin" srcip="192.168.1.2" sid="8ad7bbf2781b006d99176eea9050694811e745e04acfab3dd0179620109a41ab" facility="webadmin" client="webadmin.plx" call="new"<31>May 23 00:19:33 confd[41177]: D sys::AUTOLOAD:307() => id="3100" severity="debug" sys="System" sub="confd" name="external call" user="admin" srcip="192.168.1.2" facility="webadmin" client="webadmin.plx" lock="none" method="get_SID"
2022:05:23-00:50:24 test confd[5198]: I Session::terminate:292() => id="3100" severity="info" sys="System" sub="confd" name="closing session" user="admin" srcip="192.168.1.2" sid="8ad7bbf2781b006d99176eea9050694811e745e04acfab3dd0179620109a41ab" facility="webadmin" client="webadmin.plx" call="logout" function="logout"
```

从中提取出：

- authentication successful： 2022:05:23-00:19:33
- User： admin
- srcip： 192.168.1.2
- closing session： 2022:05:23-00:50:24

将以上信息同Web管理页面Management的Last WebAdmin Sessions进行比对，发现数据保持一致

删除以上信息：

```
sed -i "/8ad7bbf2781b006d99176eea9050694811e745e04acfab3dd0179620109a41ab/d" /var/log/confd.log
```

刷新Web管理页面Management，发现该方法无法清除Last WebAdmin Sessions记录

### 8.编辑数据库中存储的Session信息

查询指定sid的信息：

```
psql reporting -U postgres -c "SELECT sid,facility,srcip,username,time,endtime,state FROM confd_sessions WHERE sid ='8ad7bbf2781b006d99176eea9050694811e745e04acfab3dd0179620109a41ab';"
```

删除指定sid的信息：

```
psql reporting -U postgres -c "DELETE FROM confd_sessions WHERE sid ='f7cce7739e98229816be6b186ada2e2942064cbf0093e329e98939fe65d8d3e3';"
```

刷新Web管理页面Management，发现该方法能够清除Last WebAdmin Sessions记录（包括Changelog）

## 0x04 实现方法
---

综合以上内容，得出清除Last WebAdmin Sessions记录的方法：清除数据库`reporting`中对应的记录

具体方法如下：

### 1.确认Last WebAdmin Sessions记录对应的sid

读取文件`/var/log/confd.log`，查询命令：

```
cat /var/log/confd.log| grep success
```

从返回结果中确认Session记录的sid

### 2.删除sid对应的Session记录

命令示例：

```
psql reporting -U postgres -c "DELETE FROM confd_sessions WHERE sid ='f7cce7739e98229816be6b186ada2e2942064cbf0093e329e98939fe65d8d3e3';"
```

## 0x05 小结
---

本文介绍了清除Last WebAdmin Sessions记录的细节。



---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)


