PG 的编译方法还是比较简单的： 
就是configure && make && make install

但是，这不是最好的办法，这会导致编译的临时文件和代码混在一起，最好的办法就是在源码目录上层创建一个build目录，然后cd到build目录，执行，例如：
../postgres/configure --enable-debug CFLAGS="-O0 -g" --prefix=$HOME/postgresql/pg_install

然后就会在当前build目录创建编译环境，然后就可以 make && make install





