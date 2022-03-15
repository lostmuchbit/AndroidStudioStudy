```git
git config --global user.name "Tony" 设定用户名

git config --global user.email "tony@gmail.com"  设定邮箱

git init  初始化git仓库

ls -al 命令查看当前文件夹下所有的文件和文件夹

git add .  把想要提交的文件加到更改里面

git commit .  -m  "Msg"   提交到本地仓库  -m后面是本次提交的信息

.gitignore文件     配置忽略文件

git status     查看有哪些文件被修改过了

git diff  A.kt   查看A.kt这个文件被修改了什么

git checkout A.kt   撤销A.kt中修改了但是还未提交的内容(add 过了就不能checkout了)

git reset A.kt   已经add过了的A.kt中的修改(commit 过了的不能reset)，可以撤回添加，然后再checkout撤销修改

git log 查看历史提交信息

git log id   可以查看指定id 的 记录

git log -1   查看最后一次的提交记录
12
```