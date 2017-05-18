目前项目模块化工作已经完成，整个项目分为四大业务模块和一个底层，然后每个业务模块都指向公司内网的git私有库，平常开发小组都会注释掉其他模块，缩减编译时间。现在问题来了，一旦需要更新其他模块的代码时，pod install也不会即使更新。

先来介绍pod引用私有库的两种方式：

1. 引用.git库

```
pod 'BFRefreshHeader', :git => 'http://.../ios_components/BFRefreshHeader.git', :tag => '1.0' 


```
>此处引用公司内网的.git地址，pod install会出现在Pods项目下的Pods文件下，所有的库文件在一个group下无序的显示，没有开发时的group层级关系，不适合开发，适合不需要经常改动的组件库。需要在执行pod install更新代码时必须指向新的tag。

2. 引用本地库

```
pod 'BFRefreshHeader', :path => '../../BFRefreshHeader'

```
>此处需先将1中的.git地址clone到本地，然后将path（注意：1中是git）指向本地路径，pod install会出现在Pods项目下的Development Pods文件下，所有的库文件按正常的层级group显示，适合开发，适合经常改动的组件库。更新代码需要在模块项目中执行git pull，然后在主项目中执行pod install。

综上所诉，当需要跨模块联调时，需要更新对方模块代码，不可能让对方打一个tag给你更新。所以我们采用第二种方式引用。将四个模块全部clone到本地，然后只在联调的时候我们才引入主项目。

以上介绍完毕，开始本文需要解决的问题，上文提到第二种引用方式更新所有代码的流程：四个模块执行git pull——>更新Podfile为本地路径引用——>主项目pod install——>重新打开主项目（因为pod install后pod项目会打不开，需重新打开主项目，未知原因）

所以现在问题又来了，如此更新一次代码未免太过复杂繁琐，然而计算机最擅长做的不就是重复又繁琐的事情吗，所以现在写一个脚本来实现上诉流程。

- 更新的主要代码:

```
# profile文件目录
ROOT_PATH = os.getcwd()

# 业务模块名
MODULES_NAMES = ['ModuleA', 'ModuleB', 'ModuleC', 'ModuleD', 'Core']
# 本地路径名
MODULES_PATHS = ['../../ModuleA', '../../ModuleB', '../../ModuleC', '../../ModuleD', '../../Core']
# 项目远程地址
MODULES_URLS = ['http://.../ModuleA.git', 'http://.../ModuleB.git', 'http://.../ModuleC.git', 'http://.../ModuleD.git', 'http://.../Core.git']
# 最新的分支（与模块名列表对应）
MODULES_BRANCHS = ['release1.0', 'release1.0', 'release1.0', 'release1.0', 'release1.0']

def update_project():

    # 关闭Xcode
    os.system("killall Xcode")

    # 批量更新业务模块
    for path in MODULES_PATHS:
        index = MODULES_PATHS.index(path)
        sub_moudle_dir = os.path.join(ROOT_PATH, path)
        if os.path.exists(sub_moudle_dir):
            print "start update >> %s" % MODULES_NAMES[index]
            os.system("cd %s;git -C %s pull" % (ROOT_PATH, path))
        else:
            print "start clone >> %s" % MODULES_NAMES[index]
            os.system("cd %s;git clone %s %s;cd %s;git checkout -b %s origin/%s" % (ROOT_PATH, MODULES_URLS[index], path, path, MODULES_BRANCHS[index], MODULES_BRANCHS[index]))

    print "start update >> 主项目"
    # 更新主目录(注意：无主项目修改权限者禁止提交，所以会先重置主项目修改，否则主项目有修改请先提交，不然后果自负)
    os.system("cd %s;git reset --hard;git clean -d -fx"";git pull;git status" % ROOT_PATH)

    # 更新podfile，替换为本地路径
    update_podfile()

    # pod install
    os.system("cd %s;pod install" % ROOT_PATH)

    # 打开Xcode,(Xcode与应用里当前xcode名相同)
    os.system("cd %s;open -a Xcode example.xcworkspace/" % ROOT_PATH)
```

- 更新Podfile文件

```
# 更新Podfile文件
def update_podfile():
    podfile_path = ROOT_PATH + "/Podfile"

    fo = open(podfile_path,'r+')
    content = fo.read()
    for eachline in open(podfile_path,'r+'):
        pattern = re.compile(r'\'(\w+)\', :(\w+) => (.+)')
        match = pattern.search(eachline)
        if match:
            module_name = match.group(1)
            if module_name in MODULES_NAMES:
                index = MODULES_NAMES.index(module_name);
                type_name = match.group(2)
                if not (type_name == 'path'):
                    eachline_new = "  pod '%s', :path => '%s'\n" % (module_name, MODULES_PATHS[index])
                    content = content.replace(eachline, eachline_new)

    # 清空原字文件，写入替换后的内容
    fo = open(podfile_path, 'w')
    fo.write(content)
    fo.flush()
```
>此处可能有人任然用第一种.git的方式引用，或者第二种方式引用的路径不同，所以此处要替换成自己本地clone的子模块路径。

学习python不久，发现python的正则运用实在是太强大了，忍不住要写脚本的冲动。每天上班打开电脑第一件事就是执行一遍脚本，然后优雅的端起茶杯的感觉太惬意了😄😄。上面有不对的地方或可以优化的地方还望大家帮我指出，十分感谢！！！