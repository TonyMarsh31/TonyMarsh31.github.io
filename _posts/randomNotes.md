## tmux && alacritty 中的复制模式

在tmux中，复制模式是通过`prefix + [`进入的。其类似于vim中的visualmode，可以方便地进行文字选择，然后使用enter进行复制。 文字内容除了可以通过terminal emulator中已配置的paste按键绑定进行粘贴，tmux也提供了与其对应的`prefix + ]`键位进行粘贴。

而在Alacritty中，其也提供了类似的功能，使用`ctrl + space`进入复制模式，然后使用自定义的按键绑定进行各类操作。

对我来说两者的使用场景一致。tmux的scope是本会话本窗格的内容，Alacritty自带的scope是该alacritty实例window上的所有内容(应该有一个maxline配置)，因为tmux的渲染问题，使用Alacritty自带的visual模式会展现出很多你也许不需要的内容。一个典型的问题是，在tmux中即使使用了clear命令后，Alacritty的visualmode中仍然有老的内容。所以我更倾向于使用tmux的复制模式。
