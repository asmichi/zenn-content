---
title: "fork-execとFDリーク"
emoji: "🛢️"
type: "tech"
topics: ["unix"]
published: true
---

> ["everything is a file descriptor or a process"](https://lore.kernel.org/lkml/Pine.LNX.4.44.0206091056550.13459-100000@home.transmeta.com/)

--- Linus Torvalds

# はじめに

本文書は、[Asmichi.ChildProces](https://github.com/asmichi/ChildProcess)特に[ChildProcess.Native](https://github.com/asmichi/ChildProcess/tree/master/src/ChildProcess.Native)の実装中に得られた知見に関するメモです。

# fork-execとFDリーク

端的に言うと、`fork`-`exec`するとき、子プロセスにFDが継承されてしまってFDに紐づくリソースを解放できなくなることがあります。マルチスレッドのプログラムにおいて特に深刻ですが、シングルスレッドのプログラムにおいても十分に問題です。

FDに紐づくあらゆるリソースについて同じことが言えますが、以下、話を簡単にするためにパイプだけを考えます。

## パイプの書き込みの終了

実例で見ていきましょう。パイプは、書き込み側を閉じると、パイプからの読み取りはもはやブロックしなくなります。それによって書き込みが終了したことを知るのでした。

```c
// pipe.c
#include <stdio.h>
#include <unistd.h>

int main()
{
    int pipes[2];
    pipe(pipes);

    // 書き込み側を閉じる
    close(pipes[1]);

    char buf[1];
    ssize_t bytes_read;
    bytes_read = read(pipes[0], buf, sizeof(buf));

    // "0 bytes read" が出力される
    printf("%zd bytes read\n", bytes_read);

    return 0;
}
```
```
$ gcc -std=c99 -Wextra pipe.c && ./a.out
0 bytes read
```

## fork-execしてみる

パイプを生成した後、`close`より前に`fork`して`sleep 100`を実行してみましょう。

`fork`は、現在のプロセスの複製を子プロセスとして生成します。この際、現在のプロセスが持つすべてのFDも複製されます。パイプのFDの複製を子プロセスが持っていますから、もはや親プロセスだけではパイプの書き込み側を閉じることができなくなってしまいました。

```c
// pipe-and-fork.c
#include <stdio.h>
#include <unistd.h>

int fork_and_exec_sleep100();

int main()
{
    int pipes[2];
    pipe(pipes);

    // FDが子プロセスに継承される
    fork_and_exec_sleep100();

    // 書き込み側を閉じようとするが、
    // 子プロセスがこのFDの複製を持っているので閉じることができない
    close(pipes[1]);

    char buf[1];
    ssize_t bytes_read;
    // 子プロセスが終了するまで、このreadは返らない
    bytes_read = read(pipes[0], buf, sizeof(buf));

    printf("%zd bytes read\n", bytes_read);

    return 0;
}

int fork_and_exec_sleep100()
{
    int child_pid = fork();
    if (child_pid == 0)
    {
        // 子
        char* const argv[] = { "/usr/bin/sleep", "100", NULL };
        execv(argv[0], argv);
        _exit(1);
    }
    else
    {
        // 親
        return child_pid;
    }
}
```

```
$ gcc -std=c99 -Wextra pipe-and-fork.c && ./a.out
^Z
[1]+  Stopped                 ./a.out
$ killall /usr/bin/sleep
$ fg
./a.out
0 bytes read
```

# どうするのか？

もちろん、子プロセスが存在を知る由もないFDを子プロセスに継承させたのが問題です。だから、標準入出力だけを継承させればよいわけです。

以下のような方法があります。

- (Linux) close-on-exec (`FD_CLOEXEC`)
- (macOS) `POSIX_SPAWN_CLOEXEC_DEFAULT`
- (*BSD) `closefrom`

## (Linux) close-on-exec (`FD_CLOEXEC`)

close-on-exec (`FD_CLOEXEC`)フラグを持つFDは、`exec`を実行した際にクローズされます。マルチスレッドプログラムにおいては、FDの生成とアトミックにclose-on-execフラグを付与する必要がありますから、それが可能なAPIを使用する必要があります。パイプについては`pipe2`関数を使用します。

※macOS(少なくとも10以前)においては、この手法は適用できません。`pipe2`等がないためです。

```c
// pipe-and-fork-cloexec.c
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int fork_and_exec_sleep100();

int main()
{
    int pipes[2];
    // close-on-exec(`FD_CLOEXEC`)フラグ付きでパイプを生成する
    pipe2(pipes, O_CLOEXEC);

    // pipe2を使用すればアトミックにclose-on-execフラグを付与できる。
    // マルチスレッドプログラムでは必須である。
    //
    // シングルスレッドプログラムであれば、以下のようにして、
    // 生成したあとでclose-on-execフラグを付与してもよい。
    //
    // pipe(pipes);
    // fcntl(pipes[0], F_SETFD, FD_CLOEXEC);
    // fcntl(pipes[1], F_SETFD, FD_CLOEXEC);

    // close-on-execフラグが付与されているので、exec時にFDは閉じられる
    fork_and_exec_sleep100();

    // 書き込み側を閉じることができる
    close(pipes[1]);

    char buf[1];
    ssize_t bytes_read;
    bytes_read = read(pipes[0], buf, sizeof(buf));

    printf("%zd bytes read\n", bytes_read);

    return 0;
}

int fork_and_exec_sleep100()
{
    int child_pid = fork();
    if (child_pid == 0)
    {
        // 子
        char* const argv[] = { "/usr/bin/sleep", "100", NULL };
        execv(argv[0], argv);
        _exit(1);
    }
    else
    {
        // 親
        return child_pid;
    }
}
```

```
$ gcc -std=c99 -Wextra pipe-and-fork-cloexec.c && ./a.out
0 bytes read
$ killall /usr/bin/sleep
```

## (macOS) POSIX_SPAWN_CLOEXEC_DEFAULT

`posix_spawnattr_t`に`POSIX_SPAWN_CLOEXEC_DEFAULT`を付与すると、exec時にデフォルトですべてのFDがクローズされます。デフォルトでクローズされますから、必要なFDは明示的にファイルアクションで複製する必要があります。

余談ですが、`POSIX_SPAWN_SETEXEC`を付与すると、`posix_spawn`は、`fork`ではなく`exec`相当の動作になります。

いずれもmacOS拡張です。

```c
// pipe-and-fork-cloexec-default.c
#include <spawn.h>
#include <stdio.h>
#include <sys/errno.h>
#include <unistd.h>

extern char** environ;
void spawn_sleep100();

int main()
{
    int pipes[2];
    pipe(pipes);

    // POSIX_SPAWN_CLOEXEC_DEFAULTにより、exec時にFDは閉じられる
    spawn_sleep100();

    // 書き込み側を閉じることができる
    close(pipes[1]);

    char buf[1];
    ssize_t bytes_read;
    bytes_read = read(pipes[0], buf, sizeof(buf));

    printf("%zd bytes read\n", bytes_read);

    return 0;
}

void spawn_sleep100()
{
    posix_spawnattr_t attr;
    posix_spawn_file_actions_t actions;
    posix_spawnattr_init(&attr);
    posix_spawn_file_actions_init(&actions);

    posix_spawnattr_setflags(&attr, POSIX_SPAWN_CLOEXEC_DEFAULT);

    // デフォルトですべてのFDをクローズするので、必要なFDは明示的にファイルアクションで複製する必要がある。
    posix_spawn_file_actions_adddup2(&actions, STDIN_FILENO, STDIN_FILENO);
    posix_spawn_file_actions_adddup2(&actions, STDOUT_FILENO, STDOUT_FILENO);
    posix_spawn_file_actions_adddup2(&actions, STDERR_FILENO, STDERR_FILENO);

    int pid;
    char* const argv[] = { "/bin/sleep", "100", NULL };
    int err = posix_spawn(&pid, argv[0], &actions, &attr, argv, environ);
    if (err != 0)
    {
        errno = err;
        perror("posix_spawn");
    }

    posix_spawn_file_actions_destroy(&actions);
    posix_spawnattr_destroy(&attr);
}
```
```
% cc -std=c99 -Wextra pipe-and-fork-cloexec-default.c && ./a.out
0 bytes read
```

## (*BSD) closefrom

(未確認) *BSDにおいては、[closefrom(int lowfd)](https://www.freebsd.org/cgi/man.cgi?query=closefrom&sektion=2&manpath=FreeBSD+13.1-RELEASE+and+Ports)を使用して`lowfd`以上の番号を持つFDをすべてクローズすることができるそうです。

# 付録: Windows

Windowsにおいては、[PROC_THREAD_ATTRIBUTE_HANDLE_LIST](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-updateprocthreadattribute)を通じて、指定したハンドルだけを子プロセスに継承させることができます。
