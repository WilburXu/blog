# dockerfile 多阶段构建

```sh
FROM openresty/openresty:1.13.6.2-2-centos

ARG RESTY_YUM_REPO="http://mirrors.aliyun.com/repo/Centos-7.repo"
ARG WORK_DIR="/usr/local/openresty/nginx"
ARG USER=root

RUN sed -i 's/plugins=1/plugins=0/' /etc/yum.conf
RUN sed -i 's/enabled=1/enabled=0/' /etc/yum/pluginconf.d/fastestmirror.conf
RUN rm -rf /etc/yum.repos.d/CentOS*
RUN yum clean all \
    && yum-config-manager --add-repo ${RESTY_YUM_REPO} \
    && yum makecache \
    && rpm --rebuilddb \
    && yum install -y vim net-tools iproute \
    && yum groupinstall -y 'Development Tools'

RUN luarocks --tree=${WORK_DIR}/luarocks install lua-cjson \
    && luarocks --tree=${WORK_DIR}/luarocks install penlight \
    && luarocks --tree=${WORK_DIR}/luarocks install version \
    && luarocks --tree=${WORK_DIR}/luarocks install lua-resty-http \
    && luarocks --tree=${WORK_DIR}/luarocks install luaunit \
    && luarocks --tree=${WORK_DIR}/luarocks install ldoc \
    && luarocks --tree=${WORK_DIR}/luarocks install lua-discount \
    && luarocks --tree=${WORK_DIR}/luarocks install serpent \
    && luarocks --tree=${WORK_DIR}/luarocks install luacov \
    && luarocks --tree=${WORK_DIR}/luarocks install cluacov \
    && luarocks --tree=${WORK_DIR}/luarocks install mmdblua \
    && luarocks --tree=${WORK_DIR}/luarocks install lua-resty-jit-uuid \
    && luarocks --tree=${WORK_DIR}/luarocks install luasocket

FROM openresty/openresty:1.13.6.2-2-centos
ARG WORK_DIR="/usr/local/openresty/nginx"
ARG USER=root
COPY --from=0 ${WORK_DIR}/luarocks ${WORK_DIR}/luarocks

ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${WORK_DIR}/c_lib
ADD nginx/lib ${WORK_DIR}/lib
ADD nginx/c_lib ${WORK_DIR}/c_lib
ADD bin/go-dnsmasq /usr/local/bin/go-dnsmasq
ADD    nginx/conf/nginx.conf.template ${WORK_DIR}/conf/nginx.conf.template
ADD    nginx/conf/vhosts ${WORK_DIR}/conf/vhosts
ADD nginx/lua ${WORK_DIR}/lua

WORKDIR    ${WORK_DIR}

ADD entrypoint.sh /entrypoint.sh
ENTRYPOINT /entrypoint.sh
```