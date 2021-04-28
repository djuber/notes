# Chrome in docker

```text
bash-5.0$ chromium-browser       
Failed to move to new namespace: PID namespaces supported, Network namespace supported, but failed: errno = Operation not permitted
[50:50:0428/173707.189424:FATAL:zygote_host_impl_linux.cc(190)] Check failed: ReceiveFixedMessage(fds[0], kZygoteBootMessage, sizeof(kZygoteBootMessage), &boot_pid). 
#0 0x5637f4fb6049 base::debug::CollectStackTrace()
#1 0x5637f4f19536 base::debug::StackTrace::StackTrace()
#2 0x5637f4f2c10c logging::LogMessage::~LogMessage()
#3 0x5637f4f2d462 logging::LogMessage::~LogMessage()
#4 0x5637f342335e content::ZygoteHostImpl::LaunchZygote()
#5 0x5637f4ee04be content::(anonymous namespace)::LaunchZygoteHelper()
#6 0x5637f27e175b content::ZygoteCommunication::Init()
#7 0x5637f27e1ed0 content::CreateGenericZygote()
#8 0x5637f4ee2686 content::ContentMainRunnerImpl::Initialize()
#9 0x5637f4edf6fb content::RunContentProcess()
#10 0x5637f4edff62 content::ContentMain()
#11 0x5637f1bc31a5 ChromeMain
#12 0x7f118b9711a2 __libc_start_main
#13 0x5637f1bc2fee _start

bash-5.0$ chromium-browser --help
[0428/173655.978188:FATAL:chrome_main_delegate.cc(351)] execlp failed: No such file or directory (2)
#0 0x564a59197049 base::debug::CollectStackTrace()
#1 0x564a590fa536 base::debug::StackTrace::StackTrace()
#2 0x564a5910d10c logging::LogMessage::~LogMessage()
#3 0x564a55da58c1 ChromeMainDelegate::BasicStartupComplete()
#4 0x564a590c3142 content::ContentMainRunnerImpl::Initialize()
#5 0x564a590c06fb content::RunContentProcess()
#6 0x564a590c0f62 content::ContentMain()
#7 0x564a55da41a5 ChromeMain
#8 0x7f28f79391a2 __libc_start_main
#9 0x564a55da3fee _start
```

This puts a damper on running selenium tests :\)

From what I can tell this might be due to my running a debian host \(where user namespace maybe disabled - see `man 7 user_namespaces` for info on what this is\). 

