### 代码量统计脚本

```shell
git log --format='%aN' | sort -u | while read name; do echo -en "$name\t"; git log --since==2020-01-01 --until==2021-06-03  --author="$name" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 + $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }' -; done 
```

