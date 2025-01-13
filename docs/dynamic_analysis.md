# 动态分析
## sanitizer
### address sanitizer
-fsanitize=address
/fsanitize=address
### leak sanitizer
-fsanitize=leak
### undefine behavior sanitizer
### thread sanitizer 
-fsanitize=thread
## valgrind
### memcheck
valgrind --tools=memcheck -s ./executable arg1 arg2 ...
### helgrind
valgrind --tools=helgrind -s ./executable arg1 arg2 ...
