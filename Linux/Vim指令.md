#### 全局替换指令

    # 全局将a替换成为g
    :%s/a/b/g

#### 将每行第一个替换

    # 每行第一个a替换成b
    :%s/a/b

#### 将当前行第一个a替换为b

    :s/a/b/

#### 指定行替换

    # 将1到3行的a替换成为b
    :1,3s/a/b/

#### 显示行号

    :set nu

#### 复制行和粘贴

    光标移动到某一行，然后执行yy，表示复制
    光标移动到目标行，执行p，表示粘贴

#### 移动光标到行尾
    
    执行$或者￥

#### 移动光标到行首

    执行0

#### 移动到文件最低

    shift + G
