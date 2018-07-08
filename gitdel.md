### 首先通过rev-list来找到仓库记录中的大文件：
> git rev-list --objects --all | grep "$(git verify-pack -v .git/objects/pack/*.idx | sort -k 3 -n | tail -5 | awk '{print$1}')"
### 然后通过filter-branch来重写这些大文件涉及到的所有提交（重写历史记录）：
> 	git filter-branch -f --prune-empty --index-filter 'git rm -rf --cached --ignore-unmatch your-file-name' --tag-name-filter cat -- --all

### verify-pack命令用来验证Git打包的归档文件，我们用它来找到那些大文件
> git verify-pack -v .git/objects/pack/*.idx
### 输出的第一列是文件ID，第二列表示文件（blob）或目录（tree），第三列是文件大小。 现在得到了所有的文件ID及其大小，需要写一点Bash了！

### 先按照第三列排序，并取最大的5条，然后打印出每项的第一列（这一列是文件ID）：
> git verify-pack -v .git/objects/pack/*.idx | sort -k 3 -n | tail -10 | awk '{print$1}'

### 前面我们通过rev-list得到了文件名-ID的对应关系，通过verify-pack得到了最大的5个文件ID。 用后者筛选前者便能得到最大的5个文件的文件名：
> git rev-list --objects --all | grep "$(git verify-pack -v .git/objects/pack/*.idx | sort -k 3 -n | tail -5 | awk '{print$1}')"
### 只输出文件名
> git rev-list --objects --all | grep "$(git verify-pack -v .git/objects/pack/*.idx | sort -k 3 -n | tail -5 | awk '{print$1}')" | awk '{print$2}'

src/LpySever/LpySever_linux_linux
src/LpySever/go_build_LpySever.exe
src/golang.org.rar
src/server/server
src/server/server.exe
git filter-branch -f --prune-empty --index-filter 'git rm -rf --cached --ignore-unmatch src/LpySever/LpySever_linux_linux' --tag-name-filter cat -- --all

### 清理缓存
> git gc --prune=now
