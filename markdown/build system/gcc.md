静态库：

```shell
g++ -c weak.c

ar rc libweak.a weak.o
```

动态库：

```shell
g++ weak.c -o libweak.so -shared -fPIC
```

