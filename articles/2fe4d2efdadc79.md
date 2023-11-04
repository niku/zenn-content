---
title: "AWS CLI v2 で CloudWatch Logs の Log group 名一覧を取得する方法"
emoji: "😸"
type: "tech"
topics:
  - "aws"
  - "cli"
published: true
published_at: "2021-11-24 22:20"
---

# TL;DR

`jq` が使えるなら前者、使えないなら後者がよさそう

* `aws logs describe-log-groups | jq .logGroups[].logGroupName` 
* `aws logs describe-log-groups --query logGroups[].logGroupName`

```shell
❯ aws logs describe-log-groups | jq .logGroups[].logGroupName
"/aws/lambda/MyStack-ServiceMyFunction7655CEBC-10V097VHAPDP6"
"/aws/lambda/MyStack-ServiceMyFunction7655CEBC-1OM7FTO5GWVT4"
"/aws/lambda/MyStack-ServiceMyFunction7655CEBC-32A06S7C1JRZ"

❯ aws logs describe-log-groups --query logGroups[].logGroupName
[
    "/aws/lambda/MyStack-ServiceMyFunction7655CEBC-10V097VHAPDP6",
    "/aws/lambda/MyStack-ServiceMyFunction7655CEBC-1OM7FTO5GWVT4",
    "/aws/lambda/MyStack-ServiceMyFunction7655CEBC-32A06S7C1JRZ"
]
```


# 解決までの道のり
* https://docs.aws.amazon.com/cli/latest/reference/logs/ を調べる
  * list という prefix のついたコマンドで一覧を取得するケースがよくある
    * `list` という prefix がついたコマンドは [list-tags-log-group](https://docs.aws.amazon.com/cli/latest/reference/logs/list-tags-log-group.html) しかない
      * これは "Lists the tags for the specified log group." と書いてあるので違いそう
    * `list` という prefix がついていないコマンドの中から探す
      * `create` / `delete` / `filter` / `get`/ `put` は違いそうなので、それ以外から探す
      * [describe-log-groups](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/logs/describe-log-groups.html) というのがそれっぽい
        * "Lists the specified log groups. You can list all your log groups or filter the results by prefix. The results are ASCII-sorted by log group name." よさそう
        * 一覧を取得できるので、各要素の `logGroupName` を取るとよい
          * `jq` がつかえるなら、一覧の JSON を自分で加工するのが簡単
          * `jq` がつかえないなら、aws cli には [--query](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-filter.html#cli-usage-filter-client-side) というパラメータで出力を加工する方法が用意されている。こちらでも可能。
            * `jq` がなくても使えるという意味では便利
            * aws cli 専用なので `jq` よりクエリを思い出すのが難しい
