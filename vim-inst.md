# vim 操作

* 将printk都注释掉，且不影响printk缩进。其中```\1```为匹配中的第一个group。

    ```
    :%s/^\(\s*\)printk/\/\/\1printk/g
    ```

* 删除不含printk的行。
    ```
    :% v/pattern/d 
    ```
    或
    ```
    :%g!/pattern/d 
    ```

* 删除包含printk的行。
    ```
    :g/pattern/d 
    ```

