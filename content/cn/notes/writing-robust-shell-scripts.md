---
title: "编写健壮的Shell脚本"
date: 2017-02-06
type: "notes"
draft: false
---

写Shell脚本应该已经成为程序员必须掌握的技能了。因为Shell脚本简单易上手的特性，在日常工作中，我们经常使用Shell脚本来自动化应用的部署测试，环境的搭建与清理等等。殊不知，Shell脚本也会有各种坑，经常导致Shell脚本因为各种原因不能正常执行成功。实际上，编写健壮可靠的Shell脚本也是有一定的技巧的，今天我们就来一一说明。

### `set -euxo pipefail`

在执行Shell脚本的时候，通常都会创建一个新的Shell，比如，当我们执行：

```
bash script.sh
```

Bash会创建一个新的Shell来执行`script.sh`，同时也默认给定了这个执行环境的各种参数。`set`命令可以用来修改Shell环境的运行参数，不带任何参数的`set`命令，会显示所有的环境变量和Shell函数。对于所有可以定制的运行参数，请查看[官方手册](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html)，我们重点介绍其中最常用的四个。


**set -x**

默认情况下，Shell脚本执行后只显示运行结果，不会展示结果是哪一行代码的输出，如果多个命令连续执行，它们的运行结果就会连续输出，导致很难分清一串结果是什么命令产生的。

`set -x`用来在运行结果之前，先输出执行的那一行命令，行首以`+`表示是命令而非命令输出，同时，每个命令的参数也会展开，我们可以清晰地看到每个命令的运行实参，这对于Shell的debug来说非常友好。

```
#!/bin/bash
set -x

v=5
echo $v
echo "hello"

# output:
# + v=5
# + echo 5
# 5
# + echo hello
# hello
```

实际上，`set -x`还有另一种写法`set -o xtrace`。

**set -u**

Shell脚本不像其他高级语言，如Python, Ruby等，Shell脚本默认不提供安全机制，举个简单的例子，Ruby脚本尝试去读取一个没有初始化的变量的内容的时候会报错，而Shell脚本默认不会有任何提示，只是简单地忽略。

```
#!/bin/bash

echo $v
echo "hello"

# output:
#
# hello
```

可以看到，`echo $v`输出了一个空行，Bash完全忽略了不存在的`$v`继续执行后面的命令`echo "hello"`。这其实并不是开发者想要的行为，对于不存在的变量，脚本应该报错且停止执行来防止错误的叠加。`set -u`就用来改变这种默认忽略未定义变量行为，脚本在头部加上它，遇到不存在的变量就会报错，并停止执行。

```
#!/bin/bash
set -u

echo $a
echo bar

# output:
# ./script.sh: line 4: v: unbound variable
```

`set -u`另一种写法是`set -o nounset`


**set -e**

对于默认的Shell脚本运行环境，有运行失败的命令（返回值非0），Bash会继续执行后面的命令：

```
#!/bin/bash

unknowncmd
echo "hello"

# output:
# ./script.sh: line 3: unknowncmd: command not found
# hello
```

可以看到，Bash只是显示有错误，接着继续执行Shell脚本，这种行为很不利于脚本安全和排错。实际开发中，如果某个命令失败，往往需要脚本停止执行，防止错误累积。这时，一般采用下面的写法：

```
command || exit 1
```

上面的写法表示只要`command`有非零返回值，Shell脚本就会停止执行。如果停止执行之前需要完成多个操作，就要采用下面三种更高级的写法：

```
command || { echo "command failed"; exit 1; }

if ! command; then echo "command failed"; exit 1; fi

command
if [ "$?" -ne 0 ]; then echo "command failed"; exit 1; fi
```

此外，我们就联想到另外一种很类似的用法，如果两个命令有继承关系，只有第一个命令成功了，才能继续执行第二个命令，那么就要采用下面的写法：

```
command1 && command2
```

但是这些技巧多少有些麻烦，容易疏忽。而`set -e`从根本上解决了这个问题，它使得脚本只要发生错误，就终止执行：

```
#!/bin/bash
set -e

unknowncmd
echo "hello"

# output:
# ./script.sh: line 4: unknowncmd: command not found
```

可以看到，第4行执行失败以后，脚本就终止执行了。

`set -e`根据命令的返回值来判断命令是否运行失败。但是，某些命令的非零返回值可能不表示失败，或者开发者希望在命令失败的情况下，脚本继续执行下去：

```
#!/bin/bash
set -e

$(ls foobar)
echo "hello"

# output:
# ls: cannot access 'foobar': No such file or directory
```

可以看到，打开`set -e`之后，即使`ls`是一个已存在的命令，但因为`ls`命令的运行参数`foobar`实际上并不存在导致命令的返回非0值，这有时候并不是我们看到的。

可以暂时关闭`set -e`，该命令执行结束后，再重新打开`set -e`：

```
#!/bin/bash
set -e

set +e
$(ls foobar)
set -e

echo "hello"

# output:
# ls: cannot access 'foobar': No such file or directory
# hello
```

上面代码中，`set +e`表示关闭`-e`选项，`set -e`表示重新打开`-e`选项。

还有一种方法是使用`command || true`，使得该命令即使执行失败，脚本也不会终止执行。

`set -e`还有另一种写法`set -o errexit`。

**set -o pipefail**

`set -e`有一个例外情况，就是不适用于管道命令。对于管道命令，Bash会把最后一个子命令的返回值作为整个命令的返回值。也就是说，只要最后一个子命令不失败，管道命令总是会执行成功，因此它后面命令依然会执行，`set -e`就失效了。

请看下面这个例子。

```
#!/bin/bash
set -e

foo | echo "bar"
echo "hello"

# output:
# ./script.sh: line 4: foo: command not found
# bar
# hello
```

可以看到，`foo`是一个不存在的命令，但是`foo | echo bar`这个管道命令还是会执行成功，导致后面的`echo hello`会继续执行。

`set -o pipefail`用来解决这种情况，只要一个子命令失败，整个管道命令就失败，脚本就会终止执行：

```
#!/bin/bash
set -e
set -o pipefail

foo | echo "bar"
echo "hello"

# output:
# ./script.sh: line 5: foo: command not found
# bar
```

可以看到，`echo hello`命令并没有执行。

**合并四个参数**

对于上面提到的四个`set`命令参数，一般都放在一起使用。

```
# 写法一
set -euxo pipefail

# 写法二
set -eux
set -o pipefail
```

这两种写法任选其一放在所有Shell脚本的头部。

当然，也可以在在执行Shell脚本的时候，从Bash命令行传入这些参数：

```
bash -euxo pipefail script.sh
```
