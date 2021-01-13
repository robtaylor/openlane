
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
ROOT_DIR:=$(shell dirname $(realpath $(firstword $(MAKEFILE_LIST))))

CACHE_ID ?= efabless
DOCKER_ID ?= $(CACHE_ID)
IMAGE_NAME ?= $(DOCKER_ID)/openlane:rc7
TARGET_ARCHS ?= amd64 arm64
BUILDX_OPTIONS ?=
CROSS_BUILD ?=
HOST_ARCH ?= $(shell uname -m)
CACHE_DIR = $(ROOT_DIR)/docker-cache

ifeq ($(HOST_ARCH),x86_64)
  HOST_ARCH = amd64
endif 

include comma-seperate.mk

TOOLS = base boost_base replace opendp route_14 opensta yosys antmicro_yosys magic openroad_app padring netgen vlog2verilog openphysyn cvc cugr

default: openlane

.PHONY: build-all-message push-all-message

ifdef CROSS_COMPILE
TOOL_BUILD_TARGETS := $(foreach arch,$(TARGET_ARCHS),$(foreach tool,$(TOOLS),build-$(tool)-$(arch)))
TOOL_PUSH_TARGETS := $(foreach arch,$(TARGET_ARCHS),$(foreach tool,$(TOOLS),push-$(tool)-$(arch)))
MULTIARCH = $(call comma-separate,$(foreach arch,$(TARGET_ARCHS),linux/$(arch)))
TOOL_TAGS := $(foreach tool,$(TOOLS),$(foreach arch,$(TARGET_ARCHS),$(CACHE_ID)/openlane-$(tool)-$(arch)))

build-all-message:
	@echo "Building tools for $(TARGET_ARCHS)"

push-all-message:
	@echo "Pushing tools for $(TARGET_ARCHS)"
else

TOOL_BUILD_TARGETS := $(foreach tool,$(TOOLS),build-$(tool)-$(HOST_ARCH))
TOOL_PUSH_TARGETS := $(foreach tool,$(TOOLS),push-$(tool)-$(HOST_ARCH))
MULTIARCH = linux/$(HOST_ARCH)
TOOL_TAGS := $(foreach tool,$(TOOLS),%(CACHE_ID)/openlane-$(tool)-$(HOST_ARCH))

build-all-message:
	@echo "Building tools for $(HOST_ARCH)"

push-all-message:
	@echo "Pushing tools for $(HOST_ARCH)"
endif

build-all: build-all-message $(TOOL_BUILD_TARGETS)
push-all: push-all-message $(TOOL_PUSH_TARGETS)

.PHONY: cache build-all
cache: build-all push-all


.PHONY: openlane
openlane:
	@echo "Building $(IMAGE_NAME) for $(MULTIARCH)"
	@mkdir -p ../logs/docker
	docker buildx build --push --platform $(MULTIARCH) -t $(IMAGE_NAME) $(TOOL_BUILD_TARGETS) --build-arg BUILDKIT_INLINE_CACHE=1 $(BUILDX_OPTIONS)  --progress plain . 2>&1 | tee ../logs/docker/openlane.build.txt

.PHONY:
docker-builder:
	@echo "Checking buildx builder"
	docker buildx inspect openlane-multiarch >/dev/null 2>&1 > /dev/null || \
		docker buildx create --name openlane-multiarch --buildkitd-flags "--oci-worker-gc=false"
	docker buildx use openlane-multiarch

$(TOOL_BUILD_TARGETS): build-% : Dockerfile docker-builder
	@mkdir -p ../logs/docker
	@rm -rf ../logs/docker/$*.build.txt
	build="$*" ; \
	target="$${build%-*}" ; \
	arch="$${build#*-}" ; \
	docker buildx build --platform linux/$$arch --cache-from "type=local,src=$(CACHE_DIR)" --cache-to "type=local,dest=$(CACHE_DIR),mode=max" --target $$target -t $(CACHE_ID)/openlane-$$build -f Dockerfile --progress plain  .. 2>&1 | tee ../logs/docker/$*.build.txt


$(TOOL_PUSH_TARGETS): push-% : Dockerfile docker-builder
	@mkdir -p ../logs/docker
	@rm -rf ../logs/docker/$*.build.txt
	build="$*" ; \
	target="$${build%-*}" ; \
	arch="$${build#*-}" ; \
	docker buildx build --platform linux/$$arch --cache-from "type=local,src=$(CACHE_DIR)" --build-arg  BUILDKIT_INLINE_CACHE=1 --target=$$target -t $(CACHE_ID)/openlane-$$build -f Dockerfile --progress plain --push ..  2>&1 | tee ../logs/docker/$*.build.txt

clean-cache:
	@rm -rf $(CACHE_DIR)
