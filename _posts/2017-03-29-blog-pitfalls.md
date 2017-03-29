---
layout: default
title: 使用jekyll在GitHub Pages上搭建博客时的一些问题
category: technical
permalink: blog-pitfalls
---
使用`GitHub`搭建博客系统有很明显的优势：过程简单教程丰富；配置简单；文章更新也简单快捷，支持`markdown`语法；最后最重要的，不需要自己购买服务器或者主机，甚至连域名也可以不要，会自动分配一个`username.github.io`的域名。当然也可以绑定自己的个性域名，比如我主要就是为了让自己的域名有个地方放。。。

虽然`GitHub`以及`Jekyll`都有很详细的教程，一步一步指引新手如何搭建博客，但我一路弄下来，还是踩了不少坑。这里主要记录下这些坑，毕竟教程官网上已经十分详细了，照着一步步来即可。

一开始到时很顺利，就是在`GitHub`上新建一个`username.github.io`的`repository`，然后添加了自定义域名，基本上此时就可以访问站点上的文件了。

然后开始使用`Jekyll`初始化站点。之前没碰过`ruby`，于是直接按教程所写安装了`Jekyll`、`bundler`等`gem`，结果就遇到了几个坑。

第一个问题是，当我第一次运行`bundle update`命令式，提示权限不足无法执行:

    There was an error while trying to write to
    `/Users/moicen/.bundle/cache/compact_index/rubygems.org.443.29b0360b937aa4d161703e6160654e47/versions`.
    It is likely that you need to grant write permissions for that path.
于是很自然地，使用`sudo`运行，但是又出现一个`warning`:

    Rubygems 2.0.14.1 is not threadsafe, so your gems will be installed one at a time. Upgrade to Rubygems 2.1.0 or higher to enable parallel gem installation.
以及一个`error`:

    Gem::Installer::ExtensionBuildError: ERROR: Failed to build gem native extension.
警告很明白，就是说`Rubygems`版本低了，建议更新版本，暂时不管。这个错误就不是很明白问题在哪，不过还好接下来还有一段很长的信息，其中重点如下:

    Building nokogiri using packaged libraries.
    Using mini_portile version 2.1.0
    checking for iconv.h... yes
    checking for gzdopen() in -lz... yes
    checking for iconv... yes
    ************************************************************************
    IMPORTANT NOTICE:

    Building Nokogiri with a packaged version of libxml2-2.9.4.

        // some info...

    Note, however, that nokogiri is not fully compatible with arbitrary
    versions of libxml2 provided by OS/package vendors.
    ************************************************************************

这个`Nokogiri`跟系统的`libxml2`版本不兼容，然后就开始报错了:

    Extracting libxml2-2.9.4.tar.gz into
    tmp/x86_64-apple-darwin16/ports/libxml2/2.9.4... OK
    Running 'configure' for libxml2 2.9.4... OK
    Running 'compile' for libxml2 2.9.4... ERROR, review
    '/Library/Ruby/Gems/2.0.0/gems/nokogiri-1.6.8.1/ext/nokogiri/tmp/x86_64-apple-darwin16/ports/libxml2/2.9.4/compile.log'
    to see what happened. Last lines are:
    ========================================================================
    unsigned short* in = (unsigned short*) inb;
                         ^~~~~~~~~~~~~~~~~~~~~
    encoding.c:815:27: warning: cast from 'unsigned char *' to 'unsigned short *'
    increases required alignment from 1 to 2 [-Wcast-align]
    unsigned short* out = (unsigned short*) outb;
                          ^~~~~~~~~~~~~~~~~~~~~~
    4 warnings generated.
    CC       error.lo
    CC       parserInternals.lo
    CC       parser.lo
    CC       tree.lo
    CC       hash.lo
    CC       list.lo
    CC       xmlIO.lo
    xmlIO.c:1450:52: error: use of undeclared identifier 'LZMA_OK'
    ret =  (__libxml2_xzclose((xzFile) context) == LZMA_OK ) ? 0 : -1;
                                                   ^
    1 error generated.
    make[2]: *** [xmlIO.lo] Error 1
    make[1]: *** [all-recursive] Error 1
    make: *** [all] Error 2
    ========================================================================
    *** extconf.rb failed ***
    Could not create Makefile due to some reason, probably lack of necessary
    libraries and/or headers.  Check the mkmf.log file for more details.  You may
    need configuration options.

    /Library/Ruby/Gems/2.0.0/gems/mini_portile2-2.1.0/lib/mini_portile2/mini_portile.rb:366:in
    `block in execute': Failed to complete compile task (RuntimeError)
    from
    /Library/Ruby/Gems/2.0.0/gems/mini_portile2-2.1.0/lib/mini_portile2/mini_portile.rb:337:in
    `chdir'
    from
    /Library/Ruby/Gems/2.0.0/gems/mini_portile2-2.1.0/lib/mini_portile2/mini_portile.rb:337:in
    `execute'
    from
    /Library/Ruby/Gems/2.0.0/gems/mini_portile2-2.1.0/lib/mini_portile2/mini_portile.rb:111:in
    `compile'
    from
    /Library/Ruby/Gems/2.0.0/gems/mini_portile2-2.1.0/lib/mini_portile2/mini_portile.rb:150:in
    `cook'
    from extconf.rb:365:in `block (2 levels) in process_recipe'
    from extconf.rb:258:in `block in chdir_for_build'
    from extconf.rb:257:in `chdir'
    from extconf.rb:257:in `chdir_for_build'
    from extconf.rb:364:in `block in process_recipe'
    from extconf.rb:263:in `tap'
    from extconf.rb:263:in `process_recipe'
    from extconf.rb:556:in `<main>'

    Gem files will remain installed in
    /Library/Ruby/Gems/2.0.0/gems/nokogiri-1.6.8.1 for inspection.
    Results logged to
    /Library/Ruby/Gems/2.0.0/gems/nokogiri-1.6.8.1/ext/nokogiri/gem_make.out

    An error occurred while installing nokogiri (1.6.8.1), and Bundler
    cannot continue.
    Make sure that `gem install nokogiri -v '1.6.8.1'` succeeds before bundling.

也就是`nokogiri`安装总是出错，官网建议先更新`rubygems`，再安装`xcode-select`:

    gem update --system
    sudo xcode-select --install

完了继续安装`nokogiri`，结果还是失败，不过这次提示很清楚：

    ERROR:  Error installing nokogiri:
        nokogiri requires Ruby version >= 2.1.0.

`Ruby`版本过低，于是更新`Ruby`，但没有命令直接更新，需要安装`rvm`，然后使用`rvm`来更新`Ruby`:

    curl -L get.rvm.io | bash -s stable

`rvm`安装完成后，可能还需要将其加入环境变量`PATH`，修改`~/.profile`或者`~/.bash_profile`即可，我这里安装时已自动更新`PATH`，但还需要执行`source ~/.profile`重新加载更新后的环境变量，然后看看是否成功：

    $ rvm -v
    rvm 1.29.1 (latest) by Michal Papis, Piotr Kuczynski, Wayne E. Seguin [https://rvm.io/]

`rvm`安装完，看看有哪些版本可以更新:

    $ rvm list known
    # MRI Rubies
    [ruby-]1.8.6[-p420]
    [ruby-]1.8.7[-head] # security released on head
    [ruby-]1.9.1[-p431]
    [ruby-]1.9.2[-p330]
    [ruby-]1.9.3[-p551]
    [ruby-]2.0.0[-p648]
    [ruby-]2.1[.10]
    [ruby-]2.2[.6]
    [ruby-]2.3[.3]
    [ruby-]2.4[.0]
    ...

下面还有什么`JRuby`、`Rubinius`、`Minimalistic ruby implementation`、`Ruby Enterprise Edition`等等，不用管了，直接更新到最新版本`2.4.0`好了，执行命令`$ rvm install 2.4.0`，更新完，继续安装`nokogiri`，执行命令`$ sudo gem install nokogiri`，成功！

但当我进入`moicen.github.io`目录时，出现提示信息：

    RVM used your Gemfile for selecting Ruby, it is all fine - Heroku does that too,
    you can ignore these warnings with 'rvm rvmrc warning ignore /Users/moicen/Repositories/moicen.github.io/Gemfile'.
    To ignore the warning for all files run 'rvm rvmrc warning ignore allGemfiles'.

    Unknown ruby interpreter version (do not know how to handle): RUBY_VERSION.

使用`$ bundle install --system`更新时出现错误:

    /Library/Ruby/Site/2.0.0/rubygems/dependency.rb:308:in `to_specs': Could not find 'bundler' (>= 0) among 22 total gem(s) (Gem::MissingSpecError)
    Checked in 'GEM_PATH=/Users/moicen/.rvm/gems/ruby-2.4.0:/Users/moicen/.rvm/gems/ruby-2.4.0@global', execute `gem env` for more information
    from /Library/Ruby/Site/2.0.0/rubygems/dependency.rb:320:in `to_spec'
    from /Library/Ruby/Site/2.0.0/rubygems/core_ext/kernel_gem.rb:65:in `gem'
    from /usr/local/bin/bundle:22:in `<main>'

尝试重新安装，`$ gem install bundle`， 然后再更新`$ bundle install --system`，结果又出现权限错误:

    There was an error while trying to write to
    `/Users/moicen/.bundle/cache/compact_index/rubygems.org.443.29b0360b937aa4d161703e6160654e47/versions`.
    It is likely that you need to grant write permissions for that path.

改用`root`权限执行，`$ sudo bundle install --system`，结果还报错:

    /Users/moicen/.rvm/rubies/ruby-2.4.0/lib/ruby/site_ruby/2.4.0/rubygems.rb:270:in `find_spec_for_exe': can't find gem bundler (>= 0.a) (Gem::GemNotFoundException)
    from /Users/moicen/.rvm/rubies/ruby-2.4.0/lib/ruby/site_ruby/2.4.0/rubygems.rb:298:in `activate_bin_path'
    from /Users/moicen/.rvm/gems/ruby-2.4.0/bin/bundle:22:in `<main>'
    from /Users/moicen/.rvm/gems/ruby-2.4.0/bin/ruby_executable_hooks:15:in `eval'
    from /Users/moicen/.rvm/gems/ruby-2.4.0/bin/ruby_executable_hooks:15:in `<main>'

这个就完全没有头绪了，网上搜了许久也没找到类似问题，而且同时发现所有的`bundle`命令都提示权限不足，而用`sudo`执行时又全都报上面这一样的错误，实在不如如何解决了，于是去请教[阿男](http://weinan.io)老师，经指点，`sudo`命令运行的都是系统默认的内置`ruby`，而不是`rvm`安装的`ruby`，因此会报错，所以此时直接运行命令就行了。但我这儿因为之前用`sudo`执行过`gem`和`bundle`命令，导致其权限系统被破坏，因此直接执行命令时提示权限不足，建议删除`.bundle`文件夹，重新安装`rvm`，照做之后果然没问题了。

到此终于解决了软件环境问题，开始操作项目，找了个主题，基本上站点成型了。此间遇到一个很弱智的问题，我换了一个主题，另外想改下布局页，于是添加了`_layouts`文件夹和`default.html`文件，效果很好，但后来不知怎么把文件夹名字改了，变成`_layout`，然后修改`default.html`后怎么都没反应，很久才发现原来文件夹名字错了。

最后遇到的一个问题时，用`bundle install`安装需要的`gem`后，有好些都有好几个版本，于是运行`jekyll build`就会失败，因为不知道该用哪个版本，这时就要卸载掉不需要的版本了，如:

    $ gem list listen
    *** LOCAL GEMS ***
    listen (3.0.8, 3.0.6)

这时注意不要卸载错了版本:

    $ gem uninstall listen 3.0.6
    Select gem to uninstall:
     1. listen-3.0.6
     2. listen-3.0.8
     3. All versions
    > 1
    You have requested to uninstall the gem:
    	listen-3.0.6
    github-pages-129 depends on listen (= 3.0.6)
    If you remove this gem, these dependencies will not be met.
    Continue with Uninstall? [yN]  y
    Successfully uninstalled listen-3.0.6

我就卸载错了，然后又重新安装，卸载另一个。但这时还可能会有一个问题，就是要卸载的版本被设定为`default gem`，这时就无法直接卸载:

    $ gem list json
    *** LOCAL GEMS ***
    json (default: 2.0.2, 1.8.6)

`json 2.0.2`被设为`default`，无法卸载:

    $ gem uninstall json
    You have requested to uninstall the gem:
    json-1.8.6

卸载选项中找不到`2.0.2`，直接在命令中指定:

    $ gem uninstall json -v '2.0.2'
    ERROR:  While executing gem ... (Gem::InstallError)
    gem "json" cannot be uninstalled because it is a default gem

这个问题也在网上搜了很久，才终于找到解决办法，找到`/Users/yourname/.rvm/rubies/ruby-{version}/lib/ruby/gems/{version}/specifications
`目录，里面有个`default`目录，里面放的就是被设为`default`的`gem`，找到要删除的文件(`xxx.gemspec`)，删除即可，或者也可以将这里面的文件全部删除，甚至连带这个`default`目录也可以删除掉，然后那些`default gem`就不存在了。

以上，终于所有问题解决，站点完成，可以随意自定义自己需要的样式，发布文章了。
