# CHANGELOG

## 20210216

将主题由 [TeXt theme](https://github.com/kitian616/jekyll-TeXt-theme) 换成 [chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 。chirpy 支持随系统切换主题（普通/暗黑），且细节做的更好，比如 TOC 有自动换行，而 TeXt theme 就会因为截断而显示不全。

## 20200808

由于 [TeXt theme](https://github.com/kitian616/jekyll-TeXt-theme) 设置文件 *_data/variables.yml* 中指定的 *font_awesome* 地址 https://use.fontawesome.com/releases/v5.0.13/css/all.css 被墙了，故需将 all.css 和相关字体添加到本地。更改详情请见[此提交](https://github.com/Huang-Libo/Huang-Libo.github.io/commit/5c578e002d4fc38b72ec8444bd6f48c36bc68714)。  

**相关资源的下载和设置**：  

在 [fontawesome](https://fontawesome.com/how-to-use/on-the-web/setup/hosting-font-awesome-yourself) 下载的[此压缩包](https://use.fontawesome.com/releases/v5.14.0/fontawesome-free-5.14.0-web.zip)。本次更新使用的版本是 `5.14.0`。  

解压后，在 **/css** 目录找 `all.css`，在 **/webfonts** 中找需要的字体。可以先将 `all.css` 添加到本地，然后再通过 Chrome 的网络调试器查看缺少哪些字体，只添加需要的字体，这样可以减小不必要的引入。   

**/webfonts** 内所有字体的大小是 **2.9M**，而我需要的字体只有 **467K**。`all.css` 的大小是 `73K`，所以本次引入的资源总大小是 **540K**。  

