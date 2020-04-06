---
layout: post
tags : [rails, rails guides, rails generator]
title: Rails generator

---

* 调用generator时，generator中的所有public方法将会按照定义顺序依次执行

* `rails generate 生成器名称 --help`

  * `desc` 方法：指定Description
  * 另一种修改Description 是在生成器同级添加文件`USAGE`


* Rails提供了生成generator的generator `rails generate generator #{g_name}` g_name可以加上斜线作为目录命名空间

  * 默认在`lib/generators/#{g_name}/` 下生成`#{g_name}_generator.rb` `USAGE` `templates/(有的版本不创建)`
  * 新建的generator继承`Rails::Generators::NamedBase` 表示使用该generator需要传一个name， 该name在generator中由方法 `file_name`获得

* 使用generator时，查找路径顺序，相对于 $LOAD_PATH, rails项目下的lib属于其中，每个gem下的lib也属于：

        rails/generators/initializer/#{g_name}_generator.rb
        generators/initializer/#{g_name}_generator.rb
        rails/generators/#{g_name}_generator.rb
        generators/#{g_name}_generator.rb

* 可以在` config/application.rb`中对`config.generators`进行generator

        g.orm             :active_record
        g.template_engine :erb
        g.test_framework  :test_unit, fixture: true

  其他可以定义举例：

  * `g.stylesheets false` 去掉样式
  * `g.test_framework  :test_unit, fixture: false` 测试去掉创建固件
  * `g.fallbacks[:shoulda] = :test_unit` 如果使用shoulda时调用的generator 无法找到，就去找test_unit

          g.test_framework  :shoulda, fixture: false
          g.stylesheets     false

          # Add a fallback!
          g.fallbacks[:shoulda] = :test_unit

* 在generator中调用其他generator`hook_for other_generator_name`

* generator获得模板除了会查找source root外，还会查找`lib/templates` (查找顺序？)



---

### Generator API


* [Thor's documentation](http://rdoc.info/github/erikhuda/thor/master/Thor/Actions.html)
* `directory(source, *args, &block)` 从source directory 复制到 root directory
* `template(source, *args, &block)` Gets an ERB template at the relative source, executes it and makes a copy at the relative destination. If the destination is not given it's assumed to be equal to the source removing .tt from the filename.
* gem
* gem_group
* add_source
* gsub_file
* application Adds a line to config/application.rb directly after the application class definition
* git
* vendor Places a file into vendor which contains the specified code
* lib Places a file into lib which contains the specified code
* rakefile Creates a Rake file in the lib/tasks directory of the application
* initializer Creates an initializer in the config/initializers directory of the application
* generate
* rake
* route
* readme

* `source_root` 指定generator的source目录，目录中放置和该generator相关的模板， 如`copy_file(source, to)`的source文件将相对于此目录

* `file_name` 单数 小写

* `class_name` 首字母大写 单数

* `plural_name` 复数 首字母小写

---

### Rails Application Templates

* 模板可以用于新建rails项目：

  * rails new blog -m ~/template.rb
  * rails new blog -m http://example.com/template.rb

* 模板也可以用于已有项目

  * rake rails:template LOCATION=~/template.rb
  * rake rails:template LOCATION=http://example.com/template.rb

---

### Template API

* `gem(*args)` 把指定gem加人Gemfile，需要自己手动bundle
* `gem_group(*names, &block)`
* `add_source(source, options = {})` 在Gemfile里加人source url
* `environment/application(data=nil, options={}, &block)` 在指定的environment 或者application文件加人一段代码

        environment 'config.action_mailer.default_url_options = {host: 'http://yourwebsite.example.com'}, env: 'production'

* `vendor/lib/file/initializer(filename, data = nil, &block)`  在相应目录下添加文件以及内容，其中`file`接收相对于` Rails.root`的相对路径
* `rakefile(filename, data = nil, &block)` 在`rake/tasks`里创建文件和内容
* `generate(what, *args)` 调用what生成器
* `run(command)` 调用系统命令
* `rake(command, options = {})` 执行rake
* `route(routing_code)` 添加路由到route.rb
* `inside(dir) { ... }` 以dir为当前目录来执行命令
* `ask(question)` 返回用户对问题的输入，cool
* `yes?(question) or no?(question)`
* `git(:command)` 执行git命令

### 参考资料

* Creating and Customizing Rails Generators & Templates <http://guides.rubyonrails.org/association_basics.html>
