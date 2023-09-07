### 修改注释
`git commit --amend xxx`

### 撤回add
git reset HEAD 
### 撤回commit
`git reset --soft HEAD^`
如果你进行了n次commit 则可以使用`git reset --soft HEAD~n`
**注意这个命令仅仅是撤回commit操作，写的代码依然保留**
> `--mixed`   
意思是：不删除工作空间改动代码，撤销commit，并且撤销git add . 操作  
这个为默认参数，`git reset --mixed HEAD^` 和 `git reset HEAD^` 效果是一样的。
`--soft`    
不删除工作空间改动代码，撤销commit，不撤销`git add .`
`--hard`  
删除工作空间改动代码**，**撤销commit，撤销`git add .`** 
注意完成这个操作后，会删除工作空间代码！！！恢复到上一次的commit状态。慎重！！！