# vim: et:sw=4:sts=4
# Copyright 2020 Efabless Corporation
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# syntax = docker/dockerfile:1.0-experimental

FROM centos:centos8 as base

RUN yum -y install dnf-plugins-core centos-release-stream
RUN yum config-manager --set-enabled powertools
RUN yum install -y epel-release

RUN yum -y install \
        autoconf \
        automake \
        binutils \
        bison \
        byacc \
        bzip2 \
        cairo-devel \
        cmake \
        csh \
        curl \
	diffutils \
        gcc-8.3.1-5.1.el8 \
        gcc-c++-8.3.1-5.1.el8 \
        flex \
        gawk \
        gdb \
        gettext \
        git \
        graphviz \
        libtool-ltdl \
	libstdc++-static \
        make \
        ninja-build \
        libX11-devel \
	libarchive \
        libffi-devel \
        ncurses-devel \
        pcre-devel \
        python36 \
        python36-devel \
        readline-devel \
        strace \
        swig \
        tcl \
        tcl-devel \
        tk \
        tk-devel \
        wget \
        which \
        zlib-devel
RUN yum -y clean all

COPY docker/stamp.sh /


################# Boost and SWIG ###################
FROM base AS boost_base

RUN wget https://dl.bintray.com/boostorg/release/1.72.0/source/boost_1_72_0.tar.bz2
RUN tar -xf /boost_1_72_0.tar.bz2 && \
    cd boost_1_72_0 && \
    ./bootstrap.sh && \
    ./b2 install --with-iostreams --with-test --with-system --with-filesystem --with-program_options --link=shared -j $(nproc) && \
    cd .. && \
    rm -rf boost_1_72.0

#RUN wget https://github.com/swig/swig/archive/rel-4.0.1.tar.gz
#RUN tar xfz rel-4.0.1.tar.gz \
#    && rm -rf rel-4.0.1.tar.gz \
#    && cd swig-rel-4.0.1 \
#    && ./autogen.sh \
#    && ./configure --prefix=/usr && make -j $(nproc) \
#    && make install \
#    && cd .. \
#    && rm -rf swig-rel-4.0.1


######### Antmicro's Yosys staging tree ###########

FROM base as antmicro_yosys
RUN git clone https://github.com/antmicro/yosys yosys && \
    cd yosys && \
    git checkout fe58e937ef87fdb157dd89c365bb6a570fe616ea

WORKDIR yosys
RUN make PREFIX=/build/antmicro config-gcc && \
    make PREFIX=/build/antmicro -j 2  && \
    make PREFIX=/build/antmicro install

RUN /stamp.sh antmicro_yosys


####################### CUGR ########################
FROM boost_base AS cugr
RUN yum -y install vim-common glibc-static spdlog-devel

#Dependencies - libdef, liblef
RUN git clone https://github.com/The-OpenROAD-Project/OpenDB.git && \
    cd OpenDB && \
    git checkout 32c13c6cd6894066b086356a7ff1f97894cafefb
WORKDIR OpenDB
RUN mkdir build && \
    mkdir /build

RUN cd build && \
    cmake .. -DCMAKE_BUILD_TYPE=release && \
    make -j $(nproc) && \
    make install 

RUN git clone https://github.com/ax3ghazy/cu-gr cugr_12012021 \
       && cd cugr_12012021 \
       && git checkout 1632b4b450cbd3e5b6bdc9bf92c96cadde6a01b9

WORKDIR cugr_12012021

RUN touch foo-$(date -Iseconds)
RUN python3 scripts/build.py -o release --cmake_options="-DBoost_NO_BOOST_CMAKE=ON"

RUN mkdir -p /build/bin/ && \
    cp run/iccad19gr /build/bin/cugr

RUN /stamp.sh cugr


####################### CVC ########################
FROM base AS cvc

RUN pip3 install pyinstaller

COPY docker/cvc/cvc-1.0.0.tar.gz cvc-1.0.0.tar.gz
RUN tar -xvf cvc-1.0.0.tar.gz
WORKDIR cvc-1.0.0
RUN ./configure --disable-nls --prefix=/build && \
    make install


############### Oh oh oh, its Magic ################
FROM base AS magic
RUN yum install -y mesa-libGLU-devel cairo-devel
RUN git clone git://opencircuitdesign.com/magic magic_18122020

WORKDIR magic_18122020
RUN git checkout a31bd9f00363c21367b60f482537ce3dd49524ce 

RUN ./configure --prefix=/build && \
    make -j$(nproc) && \
    make install

RUN /stamp.sh magic


##################### Netgen #######################
FROM base AS netgen
RUN git clone git://opencircuitdesign.com/netgen netgen_15122020
WORKDIR netgen_15122020
RUN git checkout 69780aa0483e9fff320a51c88a927551f6f3c765

RUN ./configure --prefix=/build
RUN make clean && \
    make && \
    make install

RUN /stamp.sh netgen


##################### OpenDP #######################
FROM base AS opendp

RUN git clone --recursive https://github.com/kareefardi/OpenDP
RUN mkdir /OpenDP/build
WORKDIR /OpenDP/build

RUN cmake .. && \
    make && \
    make install && \
    mkdir -p /build/bin/ && \
    cp opendp /build/bin

RUN /stamp.sh opendp


##################### OpenPhySyn #######################
FROM boost_base AS openphysyn

RUN git clone https://github.com/scale-lab/OpenPhySyn.git && \
    cd OpenPhySyn && \
    git checkout 6bb80c5f9312a302b7c29cf1a4ddb93ace47d122 && \
    git submodule update --init --recursive

WORKDIR OpenPhySyn
COPY docker/openphysyn/*.patch .
RUN cd external/OpenDB && git apply ../../opendb.cmake.patch
RUN cd external/OpenSTA/ && git apply ../../opensta.cmake.patch


RUN mkdir build && \
    mkdir /build

RUN cd build && \
    cmake .. -DCMAKE_BUILD_TYPE=release && \
    make -j $(nproc) && \
    make install 

RUN mkdir /build/bin && \
    cp /usr/local/bin/Psn /build/bin/

RUN /stamp.sh openphysyn


##################### OpenROAD #######################
FROM boost_base AS openroad_app

RUN git clone https://gitlab.com/libeigen/eigen.git \
    && cd eigen \
    && mkdir build \
    && cd build \
    && cmake .. \
    && make install

RUN cd / && wget http://lemon.cs.elte.hu/pub/sources/lemon-1.3.1.tar.gz \
    && tar -xf lemon-1.3.1.tar.gz \
    && cd lemon-1.3.1 \
    && cmake -B build . \
    && cmake --build build -j $(nproc) --target install

RUN git clone https://github.com/The-OpenROAD-Project/OpenROAD OpenROAD_23112020
WORKDIR /OpenROAD_23112020/
RUN git checkout d03ebfc2447b674583b77d94d0891e4c3bc6b1a2 && \
    git submodule update --init --recursive

COPY docker/openroad_app/rails.patch /OpenROAD_23112020/
RUN git apply rails.patch

COPY docker/openroad_app/setup_local.patch /OpenROAD_23112020/src/OpenDB/src/swig/python
RUN cd /OpenROAD_23112020/src/OpenDB/src/swig/python && git apply setup_local.patch

COPY docker/openroad_app/pdngen_export_subst.patch /OpenROAD_23112020/
RUN git apply pdngen_export_subst.patch

COPY docker/openroad_app/ignore_obs_outside.patch /OpenROAD_23112020/
RUN git apply ignore_obs_outside.patch

# Build OpenROAD
RUN mkdir build && mkdir -p /build/version && mkdir install
RUN cd build && cmake -DCMAKE_INSTALL_PREFIX=$(pwd)/install .. && make -j$(nproc)
RUN cd build && make install
RUN cp -r build/install/bin /build/

# OpenDB Python interface
WORKDIR src/OpenDB
RUN mkdir build && mkdir -p /build/version && mkdir install
RUN cd build && cmake -DCMAKE_INSTALL_PREFIX=$(pwd)/install .. && make -j$(nproc) opendbpy
RUN cp -r build/src/swig/python /build/OpenDB_python
WORKDIR /OpenROAD_23112020/

RUN /stamp.sh openroad_app


##################### OpenSTA #######################
FROM boost_base AS opensta

# download CUDD
RUN wget https://www.davidkebo.com/source/cudd_versions/cudd-3.0.0.tar.gz && \
    tar -xvf cudd-3.0.0.tar.gz && \
    cd cudd-3.0.0 && \
    ./configure && \
    make && \
    make install

RUN git clone https://github.com/The-OpenROAD-Project/OpenSTA
WORKDIR OpenSTA
RUN git checkout 6deaf6d8fcabc884063941c3046eb8bdb80061b5

RUN mkdir /build
RUN mkdir build
WORKDIR build
RUN cmake -DCMAKE_INSTALL_PREFIX=/build -DCUDD=/usr/local .. && \
    make -j$(nproc) && \
    mkdir -p /build/bin/ && \
    cp ../app/sta /build/bin

RUN /stamp.sh opensta


##################### padring #######################
FROM base as padring

RUN git clone https://github.com/ax3ghazy/padring && \
    cd padring && \
    git checkout a88faf5a4faef75ff241276599fef81c3653cb70


WORKDIR padring

RUN ./bootstrap.sh && \
    cd build/ && \
    ninja-build && \
    mkdir -p /build/bin/ && \
    cp padring /build/bin

RUN /stamp.sh padring


##################### RePlAce #######################
FROM boost_base AS replace

RUN yum install -y \
        ImageMagick \
        ImageMagick-devel \
        ctags \
        glibc-static \
        gmp-devel \
        libjpeg-turbo-static \
        libmpc-devel \
        libtool-ltdl \
        mpfr-devel \
        zlib-static
RUN yum -y clean all

RUN git clone https://github.com/The-OpenROAD-Project/RePlAce && \
    cd RePlAce && \
    git checkout 950b5df4ea6a70fcfb1d496e85cfa63bdd172499 && \
    git submodule update --init --recursive

RUN mkdir RePlAce/build
WORKDIR RePlAce/build

RUN cmake .. && \
    make -j$(nproc) && \
    mkdir -p /build/bin/ && \
    cp /RePlAce/build/replace /build/bin/replace

RUN /stamp.sh replace


##################### TritonRoute #######################
FROM boost_base as route_14

RUN git clone https://github.com/The-OpenROAD-Project/TritonRoute.git TritonRoute_26102020 \
    && cd TritonRoute_26102020 \
    && git checkout a639a8b8c99fb078057d8449aa57333952b06309

RUN mkdir TritonRoute_26102020/build

WORKDIR TritonRoute_26102020/build

RUN cmake ../ && \
    make && \
    mkdir -p /build/bin/ && \
    cp TritonRoute /build/bin/TritonRoute


RUN /stamp.sh route_14


##################### vlog2Verilog #######################
FROM base AS vlog2verilog

RUN git clone https://github.com/RTimothyEdwards/qflow
WORKDIR qflow
RUN git checkout a550469b63e910ede6e3022e2886bca96462c540

RUN ./configure && \
    cd src && \
    make vlog2Verilog && \
    mkdir -p /build/bin && \
    cp /qflow/src/vlog2Verilog /build/bin/

RUN /stamp.sh vlog2Verilog


##################### yosys #######################
FROM base AS yosys

RUN git clone https://github.com/YosysHQ/yosys yosys && \
    cd yosys && \
    git checkout 84e9fa764803e93c59aff9f10bcc1a5ae2f9f120

WORKDIR yosys

RUN make PREFIX=/build config-gcc \
 && make PREFIX=/build -j 2 \
 && make PREFIX=/build install

RUN /stamp.sh yosys


##################### klayout #######################
FROM base AS klayout

# install klayout
RUN yum install -y qt5-devel
RUN git clone https://github.com/KLayout/klayout.git && \
    cd klayout && \
    git checkout v0.26.9-2

WORKDIR klayout
RUN mkdir -p /build/lib/ /build/bin
RUN ./build.sh -rpath /build/lib/ -bin /build/bin
RUN mv /build/bin/lib*.so* /build/lib

RUN /stamp.sh klayout


############## combined tools image #################
FROM base AS tools

#TOOLS = replace opendp route opensta yosys antmicro_yosys magic openroad_app padring netgen vlog2verilog openphysyn cvc
COPY --from=replace         /build/* /
COPY --from=opendp          /build/* /
COPY --from=route_14        /build/* /
COPY --from=opensta         /build/* /
COPY --from=yosys           /build/* /
COPY --from=antmicro_yosys  /build/* /
COPY --from=magic           /build/* /
COPY --from=openroad_app    /build/* /
COPY --from=padring         /build/* /
COPY --from=netgen          /build/* /
COPY --from=vlog2verilog    /build/* /
COPY --from=openphysyn      /build/* /
COPY --from=cvc             /build/* /
COPY --from=cugr            /build/* /
COPY --from=klayout         /build/* /


##################### openlane #######################

FROM tools as openlane

RUN yum install -y \
        Xvfb \
        libXext \
        libXft \
        libgomp \
        mesa-libGLU-devel
RUN yum -y clean all

RUN pip3 install --no-cache-dir Cython & \
    pip3 install --no-cache-dir numpy==1.19.3 & \
    pip3 install --no-cache-dir \
        matplotlib \
        jinja2 \
        pandas \
        install \
        XlsxWriter

ENV OPENLANE_ROOT=/openLANE_flow
ENV OPENROAD=/build/

ENV PATH=$OPENLANE_ROOT:$OPENLANE_ROOT/scripts:$OPENROAD/bin:$OPENROAD/bin/Linux-x86_64:$OPENROAD/pdn/scripts:$PATH
ENV LD_LIBRARY_PATH=$OPENROAD/lib:$LD_LIBRARY_PATH
ENV MANPATH=$OPENROAD/share/man:$MANPATH

COPY docker/tclshrc /.tclshrc
RUN cp /.tclshrc /root
COPY . $OPENLANE_ROOT

# install opendb python
RUN cd $OPENROAD/OpenDB_python/ && \
    python3 setup.py install && \
    rm -rf $OPENROAD/OpenDB_python

WORKDIR $OPENLANE_ROOT


