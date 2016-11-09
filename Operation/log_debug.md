# 1.12 日志和调试

一般来说，你应该在运行时增加调试选项来调试问题；也可以把调试选项添加到 Ceph 配置文件里来调试集群启动时的问题，然后查看 `/var/log/ceph` （默认位置）下的日志文件。

**Tip：** 调试输出会拖慢系统，这种延时有可能掩盖竞争条件。

日志记录是资源密集型任务。如果你碰到的问题在集群的某个特定区域，只启用那个区域对应的日志功能即可。例如，你的 OSD 运行良好、元数据服务器却有问题，这时应该先打开那个可疑元数据服务器实例的调试日志；如果不行再打开各子系统的日志。

**重要：** 详尽的日志每小时可能超过 1GB ，如果你的系统盘满了，这个节点就会停止工作。

如果你要打开或增加 Ceph 日志级别，确保有足够的系统盘空间。滚动日志文件的方法见下面的 **加快日志更迭** 小节。集群稳定运行后，可以关闭不必要的调试选项以优化运行。集群在运行中记录调试输出信息会拖慢系统、且浪费资源。

### 运行时

如果你想在运行时查看某一进程的配置，必须先登录对应主机，然后执行命令：

	ceph daemon {daemon-name} config show | less

例如：

	ceph daemon osd.0 config show | less

要在运行时激活 Ceph 的调试输出（即 `dout()` ），用 `ceph tell` 命令把参数注入运行时配置：

	ceph tell {daemon-type}.{daemon id or *} injectargs --{name} {value} [--{name} {value}]

用 `osd` 、 `mon` 或 `mds` 替代 `{daemon-type}` 。还可以用星号（ `*` ）把配置应用到同类型的所有守护进程，或者指定具体守护进程的 ID 。例如，要给名为 `ods.0` 的 `ceph-osd` 守护进程提高调试级别，用下列命令：

	ceph tell osd.0 injectargs --debug-osd 0/5

`ceph tell` 命令会通过 monitor 起作用。如果你不能绑定 monitor，仍可以登录你要改的那台主机然后用 ceph daemon 来更改。例如：

	sudo ceph daemon osd.0 config set debug_osd 0/5

### 启动时

要在启动时激活调试输出（即 `dout()` ），你得把选项加入配置文件。各进程共有配置可写在配置文件的 `[global]` 段下，某类进程的配置可写在对应的守护进程段下（如 `[mon]` 、 `[osd]` 、 `[mds]` ）。例如：

	[global]
        	debug ms = 1/5

	[mon]
	        debug mon = 20
	        debug paxos = 1/5
	        debug auth = 2

	[osd]
	        debug osd = 1/5
	        debug filestore = 1/5
	        debug journal = 1
	        debug monc = 5/20

	[mds]
	        debug mds = 1
	        debug mds balancer = 1
	        debug mds log = 1
	        debug mds migrator = 1

### 加快日志更迭

如果你的系统盘比较满，可以修改 `/etc/logrotate.d/ceph` 内的日志滚动配置以加快滚动。在滚动频率后增加一个日志 `size` 选项（达到此大小就滚动）来加快滚动（通过 cronjob ）。例如默认配置大致如此：

	rotate 7
	weekly
	compress
	sharedscripts

增加一个 `size` 选项。

	rotate 7
	weekly
	size 500M
	compress
	sharedscripts

然后，打开 crontab 编辑器。

	crontab -e

最后，增加一条用以检查 `/etc/logrorate.d/ceph` 文件的语句。

	30 * * * * /usr/sbin/logrotate /etc/logrotate.d/ceph >/dev/null 2>&1

本例中每 30 分钟检查一次 `/etc/logrorate.d/ceph` 文件。

### VALGRIND 工具

调试时可能还需要追踪内存和线程问题。你可以在 Valgrind 中运行单个守护进程、一类进程、或整个集群。 Valgrind 是计算密集型程序，应该只用于开发或调试 Ceph，否则它会拖慢系统。Valgrind 的消息会记录到 `stderr` 。

### 子系统、日志和调试选项

大多数情况下，你可以通过子系统打开调试日志输出。

#### CEPH 子系统概览

各子系统都有日志级别用于分别控制其输出日志和暂存日志（内存中），你可以分别为这些子系统设置不同的记录级别。 Ceph 的日志级别从 1 到 20 ， 1 是简洁、 20 是详尽。通常，内存驻留日志不会发送到输出日志，除非：

- 致命信号出现，或者
- 源码中的 `assert` 被触发，或者
- 明确要求发送。

调试选项允许用单个数字同时设置日志级别和内存级别，这会将二者设置为一样的值。比如，如果你指定 `debug ms = 5` ， Ceph 会把日志级别和内存级别都设置为 `5` 。也可以分别设置，第一个选项是日志级别、第二个是内存级别，二者必须用斜线（ / ）分隔。假如你想把 `ms` 子系统的调试日志级别设为 `1` 、内存级别设为 `5` ，可以写为 `debug ms = 1/5` ，如下：

	debug {subsystem} = {log-level}/{memory-level}
	#for example
	debug mds log = 1/20

下表列出了 Ceph 子系统及其默认日志和内存级别。一旦你完成调试，应该恢复默认值，或一个适合平常运营的级别。

	子系统	   	      日志级别	   内存日志级别
	default			0			5
	lockdep			0			5
	context			0			5
	crush			1			5
	mds				1			5
	mds balancer	1			5
	mds locker		1			5
	mds log			1			5
	mds log expire	1			5
	mds migrator	1			5
	buffer			0			0
	timer			0			5
	filer			0			5
	objecter		0			0
	rados			0			5
	rbd				0			5
	journaler		0			5
	objectcacher	0			5
	client			0			5
	osd				0			5
	optracker		0			5
	objclass		0			5
	filestore		1			5
	journal			1			5
	ms				0			5
	mon				1			5
	monc			0			5
	paxos			0			5
	tp				0			5
	auth			1			5
	finisher		1			5
	heartbeatmap	1			5
	perfcounter		1			5
	rgw				1			5
	javaclient		1			5
	asok			1			5
	throttle		1			5

#### 日志记录选项

关于日志记录选项的详细信息，可以参考 Ceph 官方文档的对应内容 [LOGGING SETTING](http://docs.ceph.com/docs/master/rados/troubleshooting/log-and-debug/#logging-settings) 。