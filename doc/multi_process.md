# 多进程化

mongols提供的所有服务器设施既可以多线程化也可以多进程化。

并且支持在多进程化的同时多线程化。

来看代码：

```cpp

#include <unistd.h>
#include <sys/wait.h>
#include <sys/signal.h>
#include <sys/prctl.h>
#include <mongols/util.hpp>
#include <mongols/web_server.hpp>

#include <iostream>
#include <algorithm>


static void signal_cb(int sig);
static std::vector<pid_t> pids;
static void set_signal();

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
    server.set_enable_mmap(true);

    std::function<void() > process_work = [&]() {
        prctl(PR_SET_NAME, "mongols: worker");
        server.run(f);
    };
    mongols::forker(std::thread::hardware_concurrency(), process_work, pids);
    set_signal();
    for (size_t i = 0; i < pids.size(); ++i) {
        mongols::process_bind_cpu(pids[i], i);
    }

    std::function<void(pid_t) > refork = [&](pid_t pid) {
        std::vector<int>::iterator p = std::find(pids.begin(), pids.end(), pid);
        if (p != pids.end()) {
            *p = -1 * pid;
        }
        mongols::forker(1, process_work, pids);
    };
    pid_t pid;
    int status;
    while ((pid = wait(&status)) > 0) {
        if (WIFSIGNALED(status)) {
            if (WCOREDUMP(status)) {
                //std::cout << strsignal(WTERMSIG(status)) << std::endl;
                // or 
                // refork(pid);
            } else if (WTERMSIG(status) == SIGSEGV || WTERMSIG(status) == SIGBUS) {
                refork(pid);
            }
        }
    }


}

static void signal_cb(int sig) {
    switch (sig) {
        case SIGTERM:
        case SIGHUP:
        case SIGQUIT:
        case SIGINT:
            for (auto & i : pids) {
                if (i > 0) {
                    kill(i, SIGTERM);
                }
            }
            break;
        case SIGUSR1:
            for (auto & i : pids) {
                if (i > 0) {
                    kill(i, SIGSEGV);
                }
            }
            break;
        case SIGUSR2:
            for (auto & i : pids) {
                if (i > 0) {
                    kill(i, SIGSEGV);
                }
            }
            break;
        default:break;
    }
}

static void set_signal() {
    std::vector<int> sigs = {SIGHUP, SIGTERM, SIGINT, SIGQUIT, SIGUSR1, SIGUSR2};
    for (size_t i = 0; i < sigs.size(); ++i) {
        signal(sigs[i], signal_cb);
    }
}

```

多进程化最合适的场景是web_server，可以显著提升性能：


![ab_multi_process_web_server.png](image/ab_multi_process_web_server.png)

![wrk_multi_process_web_server.png](image/wrk_multi_process_web_server.png)

![nginxVSmongols.png](image/nginxVSmongols.png)

以上测试使用4个工作进程，对比于使用同样数目工作进程的nginx，更胜一筹。如果你有兴趣用更大压力测试，你会发现，比如我用wrk测试，20000并发以上，随着并发数的增加nginx的弱势会越来越明显。

## 注意

虽然mongols所包含的所有server均可以如上例所示进行多进程化从而提升性能，但是因为其中的一些server内部包含了leveldb或sqlite组件，而这两种组件只支持多线程化，并不支持多进程化，所有在对这些server进行多进程化时，`process_work`函数应该包含一些配置数据库的代码，使得不同工作进程绑定不同的数据库。

这一点非常重要。
