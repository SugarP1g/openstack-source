# oslo.config

作者：Bill\_Xiang\_

原文：https://blog.csdn.net/Bill_Xiang_/article/details/78392616

随着 OpenStack 项目的不断发展与完善， OpenStack 社区将所有组件中的具有共性的组件剥离出来，并统一放在 oslo 组件下。 oslo 中的组件不仅可以在 OpenStack 项目中使用，也可以单独作为第三方工具包供其他项目使用。 oslo.config 项目是 oslo 组件中用于 OpenStack 配置文件的一个项目。本文首先以 Nova 项目为例，介绍了 oslo.config 的用法；然后，根据源码详细分析了其实现原理。

## 1. oslo.config 用法

### 1.1 定义配置项

本小节以 nova 组件为例介绍 oslo.config 组件的用法。首先，要使用 oslo.config 需要导入该模块，一般地，直接导入 oslo.config 中的 cfg 即可。

```python
from oslo_config import cfg
```

导入 cfg 后，需要新建一个表示配置组的 OptGroup 类，如在 Nova 中添加 api 相关配置时可以添加一个 api_group 类。

```python
api_group = cfg.OptGroup('api',
    title='API options',
    help="""
Options under this group are used to define Nova API.
""")
```

api_group 表示了一组关于 api 的配置项，其中， api 表示该配置组的名称； title 描述了该配置组的标题； help 则详细描述了该配置组的作用。

创建了配置组类之后，则需要创建所有的配置项，如在 Nova 中添加关于 api 权限相关的配置参数。

```python
auth_opts = [
    cfg.StrOpt("auth_strategy",
        default="keystone",
        choices=("keystone", "noauth2"),
        deprecated_group="DEFAULT",
        help="""
This determines the strategy to use for authentication: keystone or noauth2.
'noauth2' is designed for testing only, as it does no actual credential
checking. 'noauth2' provides administrative credentials only if 'admin' is
specified as the username.
"""),
    cfg.BoolOpt("use_forwarded_for",
        default=False,
        deprecated_group="DEFAULT",
        help="""
When True, the 'X-Forwarded-For' header is treated as the canonical remote
address. When False (the default), the 'remote_address' header is used.
You should only enable this if you have an HTML sanitizing proxy.
"""),
]
```

auth_opts 配置了一组与 api 权限等相关的配置项，其中有 String 和 Bool 类型的配置项，通过分别创建 StrOpt 对象和 BoolOpt 对象设置参数的类型。

创建配置项之后，则需要将所有配置项都注册到配置组中，如在 Nova 中将 auth_opts 等配置项注册到 api_group 配置组中。

```python
API_OPTS = (auth_opts +
            metadata_opts +
            file_opts +
            osapi_opts +
            allow_instance_snapshots_opts +
            osapi_hide_opts +
            fping_path_opts +
            os_network_opts +
            enable_inst_pw_opts)
 
 
def register_opts(conf):
    conf.register_group(api_group)
    conf.register_opts(API_OPTS, group=api_group)
    conf.register_opts(deprecated_opts)
```

其中，调用 `register_opts(API_OPTS, group=api_group)` 表示将与 api 相关的配置项注册到 api_group 配置组中； `register_group(api_group)` 表示将 api_group 配置组与具体的配置文件相关联；而 `register_opts(deprecated_opts)` 则将一些不再支持的配置项单独配置。

### 1.2 读取配置文件并使用配置参数

一般， OpenStack 组件在启动服务时便加载了配置文件，读取了各配置项；因此如果想要修改的配置信息被应用到服务中需要重启服务。如 Nova 启动 nova-api 服务时，会调用 nova/cmd/api.py 中的 main() 函数，在 main() 函数中，首先便会进行读取文件的操作。

```python
config.parse_args(sys.argv)
```

而 parse_args() 函数通过 sys.argv 中指定的配置项和配置文件中的配置项，读取 api 相关的配置参数。

```python
def parse_args(argv, default_config_files=None, configure_db=True,
               init_rpc=True):
    log.register_options(CONF)
    # We use the oslo.log default log levels which includes suds=INFO
    # and add only the extra levels that Nova needs
    if CONF.glance.debug:
        extra_default_log_levels = ['glanceclient=DEBUG']
    else:
        extra_default_log_levels = ['glanceclient=WARN']
    log.set_defaults(default_log_levels=log.get_default_log_levels() +
                     extra_default_log_levels)
    rpc.set_defaults(control_exchange='nova')
    if profiler:
        profiler.set_defaults(CONF)
    config.set_middleware_defaults()
 
    CONF(argv[1:],
         project='nova',
         version=version.version_string(),
         default_config_files=default_config_files)
 
    if init_rpc:
        rpc.init(CONF)
 
    if configure_db:
        sqlalchemy_api.configure(CONF)
```

nova 通过导入 nova.conf 模块，此时会调用 nova.conf 的 \_\_init\_\_.py 文件中的各个 register_opt() 函数注册所有配置项。然后，调用了 Config 中的 CongifOpts 类的 \_\_call\_\_() 方法通过命令行参数和配置文件将配置项缓存到配置对象 CONF 中。之后，只需要在需要用到配置参数的文件中导入 CONF 对象即可使用其中的配置项。如在 Nova 中可以使用 CONF.api.enable_instance_password 读取配置文件中是否允许配置实例密码的配置项。

## 2. oslo.config实现原理

本节结合 oslo.config 的用法详细介绍 oslo.config 的实现原理。 oslo.config 项目相对比较简单，其中主要设计到 oslo_config 目录下的 cfg.py 和 types.py 等文件。其中， cfg.py 定义了配置类的数据结构和方法等； types.py 则封装了 OpenStack 中各配置项的类型。

要使用 oslo.config 最重要的便是 ConfigOpts 类，所以本文首先介绍 ConfigOpts 类。 ConfigOpts 类是一个用来提供注册配置组和配置项的配置项管理类，其中包含几个重要的属性：

- \_opts 表示所有的配置项
- \_cli\_opts 表示所有的命令行配置项
- \_groups 表示所有的配置组
- \_namespace 表示各服务的命名空间；

OpenStack 服务启动后所有的配置参数都会通过这四个参数保存到内存中。

1.2节中提到启动服务时，会调用 ConfigOpts 类的 \_\_call\_\_() 方法将服务的所有配置项缓存到内存中，其定义如下。

```python
    def __call__(self,
                 args=None,
                 project=None,
                 prog=None,
                 version=None,
                 usage=None,
                 default_config_files=None,
                 default_config_dirs=None,
                 validate_default_values=False,
                 description=None,
                 epilog=None):
        """Parse command line arguments and config files.
        Calling a ConfigOpts object causes the supplied command line arguments
        and config files to be parsed, causing opt values to be made available
        as attributes of the object.
        The object may be called multiple times, each time causing the previous
        set of values to be overwritten.
        Automatically registers the --config-file option with either a supplied
        list of default config files, or a list from find_config_files().
        If the --config-dir option is set, any *.conf files from this
        directory are pulled in, after all the file(s) specified by the
        --config-file option.
        :param args: command line arguments (defaults to sys.argv[1:])
        :param project: the toplevel project name, used to locate config files
        :param prog: the name of the program (defaults to sys.argv[0]
            basename, without extension .py)
        :param version: the program version (for --version)
        :param usage: a usage string (%prog will be expanded)
        :param description: A description of what the program does
        :param epilog: Text following the argument descriptions
        :param default_config_files: config files to use by default
        :param default_config_dirs: config dirs to use by default
        :param validate_default_values: whether to validate the default values
        :raises: SystemExit, ConfigFilesNotFoundError, ConfigFileParseError,
                 ConfigFilesPermissionDeniedError,
                 RequiredOptError, DuplicateOptError
        """
        self.clear()
 
        self._validate_default_values = validate_default_values
 
        prog, default_config_files, default_config_dirs = self._pre_setup(
            project, prog, version, usage, description, epilog,
            default_config_files, default_config_dirs)
 
        self._setup(project, prog, version, usage, default_config_files,
                    default_config_dirs)
 
        self._namespace = self._parse_cli_opts(args if args is not None
                                               else sys.argv[1:])
        if self._namespace._files_not_found:
            raise ConfigFilesNotFoundError(self._namespace._files_not_found)
        if self._namespace._files_permission_denied:
            raise ConfigFilesPermissionDeniedError(
                self._namespace._files_permission_denied)
 
        self._check_required_opts()
```

在该方法中，首先将所有配置项清空，然后根据项目名称和版本等信息获取配置文件路径，然后根据命令行参数和配置文件读取服务所有配置信息，并进行合法性校验。

除此之外， ConfigOpts 类还定义了 register_opt() 方法， register_cli_opt() 方法和 register_group() 方法等分别实现注册配置文件或命令行参数的配置项和注册配置组的操作。另外，也提供了 unregister_opt() 等方法卸载配置项，添加了 clear() 方法清空 ConfigOpts 对象中的配置信息。

在 ConfigOpts 类定义的属性和方法中，用到了两个重要的类：

- Opt 类
- OptGroup 类

其中，Opt 类定义了一个配置项的模板，主要属性包括

- 配置项名称 name
- 配置项类型 type
- 关联的 ConfigOpts 对象 dest
- 默认值 default
- 帮助信息 help 等。

为了更好的封装配置项， oslo.config 针对具体的配置参数类型为 Opt 继承了一系列子类，包括 StrOpt、BoolOpt、IntOpt、FloatOpt、ListOpt、DictOpt、IPOpt、PortOpt、HostnameOpt、HostAddressOpt、URIOpt、MultiOpt、MultiStrOpt、SubCommandOpt、_ConfigFileOpt、_ConfigDirOpt 等。

OptGroup 类则定义了一个配置组的模板，其主要属性包括：

- 配置组名称 name
- 配置组描述 title
- 配置组帮助信息 help 等。

ConfigOpts 类利用 Opt 和 OptGroup 辅助完成配置项的注册和读取等。

在读取配置文件时， oslo.config 还定义了 ConfigParser 类将配置文件中的所有配置项封装为 Json 数据包便于处理。

最后， oslo.config 将所有配置项的数据类型使用 ConfigType 类进行了封装，定义了各数据类型的类型名称、格式、最大值、最小值、最大长度等；这些数据类型主要包括 String、MultiString、Boolean、Number、Integer、Float、Port、List、Range、Dict、IPAddress、Hostname、HostAddress、URI 等。在使用时，可以调用 \_\_call\_\_() 方法校验读取的参数是否符合类型要求，也可以调用 \_formatter() 方法对配置参数进行格式化操作。