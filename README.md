
# SpirytusZ的博客源码

## 初始化

```shell
# install hexo
brew install hexo

# clone source code
git clone git@github.com:spirytusz/blog.git

# install packages
cd blog
npm install

# theme, optional
cd themes
git clone https://github.com/Kaijun/hexo-theme-huxblog.git
cp hexo-theme-huxblog/themes/huxblog huxblog
rm -rf hexo-theme-huxblog
```

## 本地调试

```shell
hexo clean # clean

hexo g # generate html documents

hexo s # launch local server
```

## 部署

```shell
hexo clean # clean

hexo d # deploy
```