# Copyright 2020 Efabless Corporation
# Copyright 2021 Rob Taylor
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

CACHE_DOCKER_ID ?= efabless
DOCKER_ID ?= $(CACHE_DOCKER_ID)
IMAGE_NAME ?= $(DOCKER_ID)/openlane:rc7
CACHE_NAME ?= $(CACHE_DOCKER_ID)/openlane:cache
BUILD_ARCHS ?= linux/amd64 linux/arm64
BUILDX_OPTIONS ?=

include comma-seperate.mk

TOOLS = base boost_base replace opendp route_14 opensta yosys antmicro_yosys magic openroad_app padring netgen vlog2verilog openphysyn cvc

TOOL_BUILD_TARGETS := $(foreach tool,$(TOOLS),build-$(tool))
tool_cache = $(foreach tool,$(TOOLS),--cache-from type=registry,ref=$(tool)$1)
TOOL_CACHES := $(call tool_cache,)
MULTIARCH = $(call comma-separate,$(BUILD_ARCHS))

TOOL_ARCH_CACHES = $(foreach arch,$(BUILD_ARCHS),$(call tool_cache,:$arch))

.PHONY: cache build-all
cache: build-all
build-all: $(TOOL_BUILD_TARGETS)

.PHONY: openlane
openlane:
	mkdir -p ../logs/docker
	docker buildx build --push --platform $(BUILD_ARCH) -t $(IMAGE_NAME) $(TOOL_CACHES) --build-arg BUILDKIT_INLINE_CACHE=1 $(BUILDX_OPTIONS)  --progress plain . 2>&1 | tee ../logs/docker/openlane.build.txt

.PHONY:
docker-builder:
	docker buildx inspect openlane-multiarch >/dev/null 2>&1 > /dev/null || \
		docker buildx create --name openlane-multiarch
	docker buildx use openlane-multiarch

$(TOOL_BUILD_TARGETS): build-% : Dockerfile docker-builder
	@mkdir -p ../logs/docker
	@rm -rf ../logs/docker/$*.build.txt

	# Loop over the architectures sepeartely to avoid potential out of memory issues (Some sort of buildx bug?)
	for i in $(BUILD_ARCH); do \
		docker buildx build --push --platform $i -t $(CACHE_DOCKER_ID)/$*:$i --cache-from type=registry,ref=$(CACHE_DOCKER_ID)/*:$i --build-arg BUILDKIT_INLINE_CACHE=1 $(BUILDX_OPTIONS) --target $* --progress plain . 2>&1 | tee -a ../logs/docker/$*.build.txt; \
	done

	#Final multiarch image assembly
	docker buildx build --push --platform $(MULTIARCH) -t $(CACHE_DOCKER_ID)/$* $(TOOL_ARCH_CACHES) --build-arg BUILDKIT_INLINE_CACHE=1 $(BUILDX_OPTIONS) --target $* --progress plain . 2>&1 | tee ../logs/docker/$*.build.txt



