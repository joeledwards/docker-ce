include ../common.mk

CLI_DIR:=$(realpath $(CURDIR)/../../cli)
ENGINE_DIR:=$(realpath $(CURDIR)/../../engine)
PLUGINS_DIR=$(realpath $(CURDIR)/../plugins)
GITCOMMIT=$(shell cd $(ENGINE_DIR) && git rev-parse --short HEAD)
STATIC_VERSION:=$(shell ../static/gen-static-ver $(ENGINE_DIR) $(VERSION))
GO_BASE_IMAGE=golang
GO_IMAGE?=$(GO_BASE_IMAGE):$(GO_VERSION)-stretch
GEN_RPM_VER=$(shell ./gen-rpm-ver $(CLI_DIR) $(VERSION))
CHOWN=docker run --rm -i -v $(CURDIR):/v -w /v alpine chown

DOCKERFILE=Dockerfile
ifdef NEEDS_ARCH_SPECIFIC
	DOCKERFILE=Dockerfile.$(ARCH)
endif
ifdef BUILD_IMAGE
	BUILD_IMAGE_FLAG=--build-arg $(BUILD_IMAGE)
endif
BUILD?=docker build \
	$(BUILD_IMAGE_FLAG) \
	--build-arg GO_IMAGE=$(GO_IMAGE) \
	-t rpmbuild-$@/$(ARCH) \
	-f $@/$(DOCKERFILE) \
	.

SPEC_FILES?=docker-ce.spec docker-ce-cli.spec
SPECS?=$(addprefix SPECS/, $(SPEC_FILES))
RPMBUILD=docker run --privileged --rm -i \
	-e PLATFORM \
	-v $(CURDIR)/rpmbuild/SOURCES:/root/rpmbuild/SOURCES \
	-v $(CURDIR)/rpmbuild/RPMS:/root/rpmbuild/RPMS \
	-v $(CURDIR)/rpmbuild/SRPMS:/root/rpmbuild/SRPMS
RPMBUILD_FLAGS?=-ba\
	--define '_gitcommit $(word 3,$(GEN_RPM_VER))' \
	--define '_release $(word 2,$(GEN_RPM_VER))' \
	--define '_version $(word 1,$(GEN_RPM_VER))' \
	--define '_origversion $(word 4, $(GEN_RPM_VER))' \
	$(SPECS)
RUN?=$(RPMBUILD) rpmbuild-$@/$(ARCH) $(RPMBUILD_FLAGS)

SOURCE_FILES=engine-image engine.tgz cli.tgz docker.service docker.socket distribution_based_engine.json plugin-installers.tgz
SOURCES=$(addprefix rpmbuild/SOURCES/, $(SOURCE_FILES))

FEDORA_RELEASES := fedora-31 fedora-30 fedora-29
CENTOS_RELEASES := centos-7

.PHONY: help
help: ## show make targets
	@awk 'BEGIN {FS = ":.*?## "} /^[a-zA-Z_-]+:.*?## / {sub("\\\\n",sprintf("\n%22c"," "), $$2);printf " \033[36m%-20s\033[0m  %s\n", $$1, $$2}' $(MAKEFILE_LIST)

.PHONY: clean
clean: ## remove build artifacts
	[ ! -d rpmbuild ] || $(CHOWN) -R $(shell id -u):$(shell id -g) rpmbuild
	$(RM) -r rpmbuild/
	[ ! -d artifacts ] || $(CHOWN) -R $(shell id -u):$(shell id -g) artifacts
	$(RM) -r artifacts/
	[ ! -d tmp ] || $(CHOWN) -R $(shell id -u):$(shell id -g) tmp
	$(RM) -r tmp/
	-docker rm docker2oci
	$(MAKE) -C ../image clean

.PHONY: rpm
rpm: fedora centos ## build all rpm packages

.PHONY: fedora
fedora: $(FEDORA_RELEASES) ## build all fedora rpm packages

.PHONY: centos
centos: $(CENTOS_RELEASES) ## build all centos rpm packages

.PHONY: $(FEDORA_RELEASES) $(CENTOS_RELEASES)
$(FEDORA_RELEASES) $(CENTOS_RELEASES): $(SOURCES)
	@echo "== Building packages for $@ =="
	$(CHOWN) -R root:root rpmbuild
	$(BUILD)
	$(RUN)
	$(CHOWN) -R $(shell id -u):$(shell id -g) rpmbuild

rpmbuild/SOURCES/engine.tgz:
	mkdir -p rpmbuild/SOURCES
	docker run --rm -i -w /v \
		-v $(ENGINE_DIR):/engine \
		-v $(CURDIR)/rpmbuild/SOURCES:/v \
		alpine \
		tar -C / -c -z -f /v/engine.tgz --exclude .git engine

rpmbuild/SOURCES/cli.tgz:
	mkdir -p rpmbuild/SOURCES
	docker run --rm -i -w /v \
		-v $(CLI_DIR):/cli \
		-v $(CURDIR)/rpmbuild/SOURCES:/v \
		alpine \
		tar -C / -c -z -f /v/cli.tgz --exclude .git cli

rpmbuild/SOURCES/docker.service: ../systemd/docker.service
	mkdir -p $(@D)
	cp $< $@

rpmbuild/SOURCES/docker.socket: ../systemd/docker.socket
	mkdir -p $(@D)
	cp $< $@

# TODO: Figure out how to decouple this
# TODO: These might just end up being static files that are hardcoded
# TODO: FROM HERE <=====================
rpmbuild/SOURCES/engine-image:
	mkdir -p $(@D)
	echo "docker.io/dockereng/engine-community-dm:$(STATIC_VERSION)" > $@

rpmbuild/SOURCES/distribution_based_engine.json: rpmbuild/SOURCES/engine-image
	mkdir -p $(@D)
	echo '{"platform":"Docker Engine - Community","engine_image":"engine-community-dm","containerd_min_version":"1.2.0-beta.1","runtime":"host_install"}' > $@
# TODO: TO HERE <=====================

rpmbuild/SOURCES/plugin-installers.tgz: $(wildcard ../plugins/*)
	docker run --rm -i -w /v \
		-v $(PLUGINS_DIR):/plugins \
		-v $(CURDIR)/rpmbuild/SOURCES:/v \
		alpine \
		tar -C / -c -z -f /v/plugin-installers.tgz --exclude .git plugins
