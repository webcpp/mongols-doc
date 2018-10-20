# 多进程化

mongols提供的所有服务器设施既可以多线程化也可以多进程化。

并且支持在多进程化的同时多线程化。

来看代码：

```cpp

#include <unistd.h>
#include <sys/wait.h>
#include <sys/signal.h>
#include <mongols/util.hpp>
#include <mongols/web_server.hpp>
#include <iostream>


static void signal_cb(int sig, siginfo_t *, void *);
static std::vector<pid_t> pids;

int main(int, char**) {
    //    daemon(1, 0);
    auto f = [](const mongols::request & req) {
        if (req.method == "GET" && req.uri.find("..") == std::string::npos) {
            return true;
        }
        return false;
    };
    int port = 9090;
    const char* host = "127.0.0.1";
    mongols::web_server
    server(host, port, 5000, 512000, 0/*2*/);
    server.set_root_path("html");
    server.set_mime_type_file("html/mime.conf");
    server.set_list_directory(true);
    server.set_enable_mmap(false);


    int child_process_len = std::thread::hardware_concurrency();
    mongols::forker(child_process_len
            , [&]() {
                server.run(f);
            }
    , pids);
    for (int i = 0; i < child_process_len; ++i) {
        mongols::process_bind_cpu(pids[i], i);
    }

    const int sig_len = 4;
    int sigs[sig_len] = {SIGHUP, SIGTERM, SIGINT, SIGQUIT};
    struct sigaction act;
    memset(&act, 0, sizeof (struct sigaction));
    sigemptyset(&act.sa_mask);
    act.sa_sigaction = signal_cb;

    for (int i = 0; i < sig_len; ++i) {
        if (sigaction(sigs[i], &act, NULL) < 0) {
            perror("sigaction error");
            return -1;
        }
    }



    for (size_t i=0;i<pids.size();++i) {
        pid_t pid;
        if ((pid = wait(NULL)) > 0) {

        }
    }
    
}

static void signal_cb(int sig, siginfo_t *, void * ) {
    switch (sig) {
        case SIGTERM:
        case SIGHUP:
        case SIGQUIT:
        case SIGINT:
            for (auto & i : pids) {
                kill(i, SIGTERM);
            }
            break;
        default:break;
    }
}

```

多进程化最合适的场景是web_server，可以显著提升性能：


![ab_multi_process_web_server.png](image/ab_multi_process_web_server.png)

![wrk_multi_process_web_server.png](image/wrk_multi_process_web_server.png)

![nginxVSmongols.png](image/nginxVSmongols.png)

以上测试使用4个工作进程，对比于使用同样数目工作进程的nginx，更胜一筹。
