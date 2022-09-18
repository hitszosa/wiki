# HITsz LUG Wiki

## 参与建设

我们希望 Wiki 的内容有正确的格式，所以我们强制要求提交的内容通过 AutoCorrect 检查。

请安装 [AutoCorrect](https://github.com/huacnlee/autocorrect) 并提交前执行以下命令：

``` shell
autocorrect --fix ./
```

Wiki 上有 [更详细的 Markdown 约定](https://wiki.hitsz.org/about/contribute-guide/markdown/)。

## 依赖

- mkdocs
- mkdocs-material

可用以下命令安装：

``` shell
pip install -r requirements.txt
```

或者如果按照了 conda 的话，推荐使用以下命令安装并激活环境：

``` shell
conda env create --name lug-wiki -f environment.yml
conda activate lug-wiki
```

## 本地预览

``` shell
mkdocs serve

# 或 mkdocs serve -a <ip:port>
```
