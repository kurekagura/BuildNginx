# ビルドガイド

Nginxとnginx-rtmp-moduleのWindows版（x86）ビルド。

【課題】

- x64は成功していない。

## ツール・バージョン

```console
> systeminfo |findstr /B /C:"OS Name" /B /C:"OS"
OS 名:                  Microsoft Windows 10 Pro
OS バージョン:          10.0.19045 N/A ビルド 19045
```

- Visual Studio Community 2022 - 17.7.0
  - Visual C++ build tools
- msys2-x86_64-20230718.exe
- strawberry-perl-5.32.1.1-64bit.msi

※「Build Tools for Visual Studio 2017」でもビルド成功を確認

インストール場所

```plaintext
O:\sw\msys64
O:\sw\Strawberry
```

## 「x86 Native Tools Command Prompt for VS 2022」

「x86 Native Tools Command Prompt for VS 2022」を開き、パスを確認しておく。

```console
> where nmake rc perl openssl
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.37.32822\bin\Hostx86\x86\nmake.exe
C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x86\rc.exe
O:\sw\Strawberry\perl\bin\perl.exe
O:\sw\Strawberry\c\bin\openssl.exe
```

## パスを通す（.bashrc）

「x86 Native Tools Command Prompt for VS 2022」プロンプトからmingw32を起動する。

```console
O:\sw\msys64\mingw32.exe
```

printenv LIB INCLUDE PATHとすると環境変数が引き継がれているのが分かる。但し、PATHは引き継がれない。.bashrcをnotepadで開き、以下を追加する（反映させるためmingw32を開きなおす）。

```plaintext
export PATH="/o/sw/Strawberry/perl/bin":${PATH}
export PATH="/o/sw/Strawberry/c/bin":${PATH}
export PATH="/C/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.37.32822/bin/Hostx86/x86:${PATH}"
export PATH="/C/Program Files (x86)/Windows Kits/10/bin/10.0.22621.0/x86:${PATH}"
```

パスを確認しておく。

```console
$ which nmake cl link rc perl openssl
/C/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.37.32822/bin/Hostx86/x86/nmake
/C/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.37.32822/bin/Hostx86/x86/cl
/C/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.37.32822/bin/Hostx86/x86/link
/C/Program Files (x86)/Windows Kits/10/bin/10.0.22621.0/x86/rc
/o/sw/Strawberry/perl/bin/perl
/o/sw/Strawberry/c/bin/openssl
```

## ソース取得

hgとgitをインストールして、ソースを取得する。

```console
pacman -S --noconfirm mercurial
pacman -S --noconfirm git

cd /o/src
hg clone http://hg.nginx.org/nginx
cd nginx

mkdir -p objs/lib
git clone https://github.com/arut/nginx-rtmp-module.git objs/lib/nginx-rtmp-module 
```

以下３つのパッケージのソースをダウンロード、/o/src/nginxに保存、objs/libへ展開する。

```console
wget https://github.com/PCRE2Project/pcre2/releases/download/pcre2-10.42/pcre2-10.42.tar.gz
wget http://zlib.net/zlib-1.3.tar.gz
wget https://www.openssl.org/source/openssl-1.1.1w.tar.gz
```

```console
tar -xzf pcre2-10.42.tar.gz -C objs/lib
tar -xzf zlib-1.3.tar.gz -C objs/lib
tar -xzf openssl-1.1.1w.tar.gz -C objs/lib
```

## ビルド

```console
auto/configure \
    --with-cc=cl \
    --with-debug \
    --prefix= \
    --conf-path=conf/nginx.conf \
    --pid-path=logs/nginx.pid \
    --http-log-path=logs/access.log \
    --error-log-path=logs/error.log \
    --sbin-path=nginx.exe \
    --http-client-body-temp-path=temp/client_body_temp \
    --http-proxy-temp-path=temp/proxy_temp \
    --http-fastcgi-temp-path=temp/fastcgi_temp \
    --http-scgi-temp-path=temp/scgi_temp \
    --http-uwsgi-temp-path=temp/uwsgi_temp \
    --with-cc-opt=-DFD_SETSIZE=1024 \
    --with-pcre=objs/lib/pcre2-10.42 \
    --with-zlib=objs/lib/zlib-1.3 \
    --with-openssl=objs/lib/openssl-1.1.1w \
    --with-openssl-opt=no-asm \
    --with-http_ssl_module \
    --add-module=objs/lib/nginx-rtmp-module
```

※ ビルドエラーが発生するので、コード修正（トラブル①）。

nmakeする。

```console
nmake
```

成功すると以下のようなメッセージが表示され、objsの下にnginx.exeが生成されている。

```console
ライブラリの検索が終了しました。
sed -e "s|%PREFIX%||"  -e "s|%PID_PATH%|/logs/nginx.pid|"  -e "s|%CONF_PATH%|/conf/nginx.conf|"  -e "s|%ERROR_LOG_PATH%|/logs/error.log|
"  < docs/man/nginx.8 > objs/nginx.8
```

## 詳細リビジョン

```console
$ hg parents
リビジョン:   9157:daf8f5ba23d8
タグ:         tip
ユーザ:       Sergey Kandaurov <pluknet@nginx.com>
日付:         Fri Sep 01 20:31:46 2023 +0400
要約:         QUIC: removed use of SSL_quic_read_level and SSL_quic_write_level.
```

```console
$ git log -n 1
commit 23e1873aa62acb58b7881eed2a501f5bf35b82e9 (HEAD -> master, tag: v1.2.2, origin/master, origin/HEAD)
Author: Roman Arutyunyan <arutyunyan.roman@gmail.com>
Date:   Tue May 25 10:42:16 2021 +0300
```

## トラブル①

```text
cl -c -O2  -W4 -WX -nologo -MT -Zi -Fdobjs/nginx.pdb -DFD_SETSIZE=1024 -DNO_SYS_TYPES_H -Yun
gx_config.h -Fpobjs/ngx_config.pch -I src/core  -I src/event  -I src/event/modules  -I src/event/qui
c  -I src/os/win32  -I objs/lib/nginx-rtmp-module  -I objs/lib/pcre2-10.42/src/  -I objs/lib/openssl
-1.1.1w/openssl/include  -I objs/lib/zlib-1.3  -I objs  -I src/http  -I src/http/modules  -Foobjs/ad
don/nginx-rtmp-module/ngx_rtmp_core_module.obj  objs/lib/nginx-rtmp-module/ngx_rtmp_core_module.c
ngx_rtmp_core_module.c
objs/lib/nginx-rtmp-module/ngx_rtmp_core_module.c(611): error C2220: 警告をエラーとして扱いました。'
object' ファイルは生成されません。
objs/lib/nginx-rtmp-module/ngx_rtmp_core_module.c(611): warning C4456: 'sa' を宣言すると、以前のロー
カル宣言が隠蔽されます
objs/lib/nginx-rtmp-module/ngx_rtmp_core_module.c(506): note: 'sa' の宣言を確認してください
NMAKE : fatal error U1077: '"C:\Program Files (x86)\Microsoft Visual Studio\2017\BuildTools\VC\Tools
\MSVC\14.16.27023\bin\Hostx86\x86\cl.EXE"' : リターン コード '0x2'
Stop.
NMAKE : fatal error U1077: '"C:\Program Files (x86)\Microsoft Visual Studio\2017\BuildTools\VC\Tools
\MSVC\14.16.27023\bin\Hostx86\x86\nmake.exe"' : リターン コード '0x2'
Stop.
```

[ngx_rtmp_core_module.cの611行目付近のsaを4箇所、sa_tmpへ変更](./workaround.patch)。
