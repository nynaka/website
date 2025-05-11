Ubuntu 24.04 で core ファイルを出力する
===

## core ファイルの出力先確認

```bash
cat /proc/sys/kernel/core_pattern
```

Ubuntu Linux 24.04 の場合、`|/usr/share/apport/apport -p%p -s%s -c%c -d%d -P%P -u%u -g%g -- %E` となっていると思います。  
この場合、Apport (Ubuntu のクラッシュレポートツール) がコアダンプを処理します。


## /tmp 配下に core ファイルを出力する

- 設定

    ```bash
    ulimit -c unlimited
    sudo sysctl -w kernel.core_pattern=/tmp/core.%e.%p
    ```

    | ファイル名形式 | 意味           |
    | :------------: | -------------- |
    |       %e       | 実行ファイル名 |
    |       %p       | プロセスID     |


- core ファイル出力確認

    - サンプルコード (crash.c)

        ```c
        #include <stdlib.h>

        int main()
        {
            abort();
            return(0);
        }
        ```

    - ビルド

        ```bash
        gcc -g -o crash crash.c
        ```

    - 実行例

        ```bash
        # サンプルコードの実行
        ./crash
        # core ファイルの出力確認
        ls -l /tmp/core.crash.*
        ```

    - gdb の実行例

        ```bash
        gdb ./crach /tmp/core.crash.848
        ```

## Makefile

```bash
NAME = crash
SRCS = crash.c
OBJS = $(SRCS:.c=.o)
CC = gcc
CFLAGS = -g -Wall -Wextra -Werror

$(NAME): $(OBJS)
	$(CC) $(CFLAGS) -o $(NAME) $(OBJS)

all: $(NAME)

.c.o:
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(NAME)

fclean: clean
	rm -f $(NAME)

re: fclean al
```
