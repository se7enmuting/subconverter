# ========= builder =========
FROM alpine:3.16 AS builder
LABEL maintainer="tindy.it@gmail.com"

ARG THREADS="4"
ARG SHA=""
# 可按需改成你要的 QuickJS 提交（必须早于 2025-04-26，且晚于 2024-01-13）
ARG QUICKJS_COMMIT="229b07b9b2c811eaf84db209a1d6f9e2a8a7b0d9"

WORKDIR /tmp

# 基础构建依赖
RUN set -xe && \
  apk add --no-cache --virtual .build-tools \
      git g++ build-base linux-headers cmake python3 pkgconf wget && \
  apk add --no-cache --virtual .build-deps \
      curl-dev rapidjson-dev pcre2-dev yaml-cpp-dev

# ---- 构建 QuickJS（开启 bignum，并固定在安全提交）----
RUN set -xe && \
  git clone https://github.com/bellard/quickjs && \
  cd quickjs && \
  git checkout "${QUICKJS_COMMIT}" && \
  make -j"${THREADS}" CONFIG_BIGNUM=y && \
  make install PREFIX=/usr CONFIG_BIGNUM=y

# ---- 构建 libcron ----
RUN set -xe && \
  git clone --depth=1 https://github.com/PerMalmberg/libcron && \
  cd libcron && \
  git submodule update --init && \
  cmake -DCMAKE_BUILD_TYPE=Release . && \
  make libcron -j"${THREADS}" && \
  install -m644 libcron/out/Release/liblibcron.a /usr/lib/ && \
  install -d /usr/include/libcron/ && \
  install -m644 libcron/include/libcron/* /usr/include/libcron/ && \
  install -d /usr/include/date/ && \
  install -m644 libcron/externals/date/include/date/* /usr/include/date/

# ---- 安装 toml11 头文件库 ----
RUN set -xe && \
  git clone --depth=1 --branch="v4.3.0" https://github.com/ToruNiina/toml11 && \
  cd toml11 && \
  cmake -DCMAKE_CXX_STANDARD=11 . && \
  make install -j"${THREADS}"

# 拷贝源码
COPY . /subconverter
WORKDIR /subconverter

# 给 quickjspp.hpp 做个兼容补丁：老版 API 名称映射
# （若目标环境已有 Unhandled 这个函数名，该宏不生效也不影响）
RUN set -xe && \
  sed -i '1i #ifndef JS_SetHostUnhandledPromiseRejectionTracker\n#define JS_SetHostUnhandledPromiseRejectionTracker JS_SetHostPromiseRejectionTracker\n#endif' include/quickjspp.hpp

# 构建 subconverter
RUN set -xe && \
  [ -n "${SHA}" ] && sed -i 's/\(v[0-9]\+\.[0-9]\+\.[0-9]\+\)/\1-'"${SHA}"'/' src/version.h; \
  python3 -m ensurepip && \
  python3 -m pip install --no-cache-dir gitpython && \
  python3 scripts/update_rules.py -c scripts/rules_config.conf && \
  cmake -DCMAKE_BUILD_TYPE=Release . && \
  make -j"${THREADS}"

# ========= runtime =========
FROM alpine:3.16
LABEL maintainer="tindy.it@gmail.com"

RUN apk add --no-cache --virtual subconverter-deps pcre2 libcurl yaml-cpp

COPY --from=builder /subconverter/subconverter /usr/bin/
COPY --from=builder /subconverter/base /base/

ENV TZ=Africa/Abidjan
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

WORKDIR /base
EXPOSE 25500/tcp
# JSON 形式 CMD，避免信号处理问题的警告
CMD ["subconverter"]
