# HITSZ 开源技术协会 Wiki

## 参与建设

我们希望 Wiki 的内容有正确的格式，所以我们强制要求提交的内容通过 AutoCorrect 检查。

请安装 [AutoCorrect](https://github.com/huacnlee/autocorrect) 并提交前执行以下命令：

``` shell
autocorrect --fix ./docs
```

Wiki 上有 [更详细的 Markdown 约定](https://wiki.hitsz.org/about/contribute-guide/markdown/)。

## 依赖

- mkdocs
- mkdocs-material

可用以下命令安装：

``` shell
# According to PEP 668, you should not use pip directly.
python -m venv venv
# Activate the venv, bash for example
source venv/bin/activate
pip install -r requirements.txt
```

For conda user:

``` shell
conda env create --name osa-wiki -f environment.yml
conda activate osa-wiki
```

## 本地预览

``` shell
mkdocs serve

# or mkdocs serve -a <ip:port>
```
