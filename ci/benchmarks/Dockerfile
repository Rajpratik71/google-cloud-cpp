# Copyright 2019 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

FROM ubuntu:bionic AS google-cloud-cpp-dependencies

RUN apt-get update && \
    apt-get --no-install-recommends install -y build-essential cmake git gcc g++ cmake \
        libc-ares-dev libc-ares2 libcurl4-openssl-dev libssl-dev make \
        pkg-config tar wget zlib1g-dev apt-utils ca-certificates apt-transport-https

# #### crc32c

WORKDIR /var/tmp/build
RUN wget -q https://github.com/google/crc32c/archive/1.0.6.tar.gz
RUN tar -xf 1.0.6.tar.gz
WORKDIR /var/tmp/build/crc32c-1.0.6
RUN cmake \
      -DCMAKE_BUILD_TYPE=Release \
      -DBUILD_SHARED_LIBS=yes \
      -DCRC32C_BUILD_TESTS=OFF \
      -DCRC32C_BUILD_BENCHMARKS=OFF \
      -DCRC32C_USE_GLOG=OFF \
      -H. -Bcmake-out/crc32c
RUN cmake --build cmake-out/crc32c --target install -- -j $(nproc)
RUN ldconfig

# #### Protobuf

WORKDIR /var/tmp/build
RUN wget -q https://github.com/google/protobuf/archive/v3.11.3.tar.gz
RUN tar -xf v3.11.3.tar.gz
WORKDIR /var/tmp/build/protobuf-3.11.3/cmake
RUN cmake \
        -DCMAKE_BUILD_TYPE=Release \
        -DBUILD_SHARED_LIBS=yes \
        -Dprotobuf_BUILD_TESTS=OFF \
        -H. -Bcmake-out
RUN cmake --build cmake-out --target install -- -j $(nproc)
RUN ldconfig

# #### gRPC

WORKDIR /var/tmp/build
RUN wget -q https://github.com/grpc/grpc/archive/78ace4cd5dfcc1f2eced44d22d752f103f377e7b.tar.gz
RUN tar -xf 78ace4cd5dfcc1f2eced44d22d752f103f377e7b.tar.gz
WORKDIR /var/tmp/build/grpc-78ace4cd5dfcc1f2eced44d22d752f103f377e7b
RUN make -j $(nproc)
RUN make install
RUN ldconfig

# #### googleapis

WORKDIR /var/tmp/build
RUN wget -q https://github.com/googleapis/cpp-cmakefiles/archive/v0.4.1.tar.gz
RUN tar -xf v0.4.1.tar.gz
WORKDIR /var/tmp/build/cpp-cmakefiles-0.4.1
RUN cmake \
    -DBUILD_SHARED_LIBS=YES \
    -H. -Bcmake-out
RUN cmake --build cmake-out --target install -- -j ${NCPU:-4}
RUN ldconfig

# #### google-cloud-cpp-common

WORKDIR /var/tmp/build
RUN wget -q https://github.com/googleapis/google-cloud-cpp-common/archive/v0.16.0.tar.gz
RUN tar -xf v0.16.0.tar.gz
WORKDIR /var/tmp/build/google-cloud-cpp-common-0.16.0
# Compile without the tests because we are testing google-cloud-cpp, not the base
# libraries
RUN cmake -H. -Bcmake-out \
    -DBUILD_TESTING=OFF \
    -DGOOGLE_CLOUD_CPP_TESTING_UTIL_ENABLE_INSTALL=OFF
RUN cmake --build cmake-out -- -j $(nproc)
RUN cmake --build cmake-out --target install
RUN ldconfig

# Get the source code

FROM google-cloud-cpp-dependencies AS google-cloud-cpp-build

WORKDIR /w
COPY . /w

# #### google-cloud-cpp

ARG CMAKE_BUILD_TYPE=Release
ARG SHORT_SHA=""

RUN cmake -H. -Bcmake-out \
    -DBUILD_TESTING=OFF \
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} \
    -DGOOGLE_CLOUD_CPP_BUILD_METADATA=${SHORT_SHA}
RUN cmake --build cmake-out -- -j $(nproc)
WORKDIR /w/cmake-out
RUN cmake --build . --target install

# ================================================================

# Prepare the final image, this image is much smaller because we only install:
# - The final binaries, without any intermdiate object files.
# - The run-time dependencies, without the build tool dependencies.
FROM ubuntu:bionic
RUN apt-get update && \
    apt-get --no-install-recommends install -y apt-utils ca-certificates apt-transport-https libc-ares2 libcurl4 \
    libstdc++6 libssl1.1 zlib1g
RUN /usr/sbin/update-ca-certificates

COPY --from=google-cloud-cpp-build /usr/local/lib /usr/local/lib
COPY --from=google-cloud-cpp-build /usr/local/bin /usr/local/bin
COPY --from=google-cloud-cpp-build /w/cmake-out/google/cloud/bigtable/benchmarks/*_benchmark /r/
COPY --from=google-cloud-cpp-build /w/cmake-out/google/cloud/storage/benchmarks/*_benchmark /r/
COPY --from=google-cloud-cpp-build /w/cmake-out/google/cloud/storage/examples/*_samples /r/
RUN ldconfig

CMD ["/bin/false"]
