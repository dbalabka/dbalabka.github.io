# dbalabka.github.io

## Install hugo
```shell
wget https://github.com/gohugoio/hugo/releases/download/v0.122.0/hugo_0.122.0_linux-amd64.deb
sudo apt install ./hugo_0.122.0_linux-amd64.deb
```
## Install theme
```shell
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git#v7.0 themes/PaperMod
git submodule update --init --recursive
```
See more details: https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-installation/

## Start server
```shell
hugo server
```