BASE_DIRECTORY=$(shell git rev-parse --show-toplevel)
RELEASE_BRANCH?=1-18
DEFAULT_RELEASE=$(shell cat $(BASE_DIRECTORY)/release/$(RELEASE_BRANCH)/RELEASE)
RELEASE?=$(or $(BUILD_ID),$(DEFAULT_RELEASE))
ARTIFACT_BUCKET?=my-s3-bucket
GIT_TAG?=$(shell cat GIT_TAG)

REPO=etcd
COMPONENT=etcd-io/$(REPO)
CLONE_URL=https://github.com/$(COMPONENT).git

DEVELOPMENT?=true
AWS_ACCOUNT_ID?=$(shell aws sts get-caller-identity --query Account --output text)
AWS_REGION?=us-west-2

MAKE_ROOT=$(shell cd "$(shell dirname "${BASH_SOURCE[0]}")" && pwd -P)
BASE_REPO?=$(AWS_ACCOUNT_ID).dkr.ecr.$(AWS_REGION).amazonaws.com
BASE_IMAGE_NAME?=eks-distro/base
BASE_TAG?=$(shell cat $(MAKE_ROOT)/../../../EKS_DISTRO_BASE_TAG_FILE)
BASE_IMAGE?=$(BASE_REPO)/$(BASE_IMAGE_NAME):$(BASE_TAG)

IMAGE_REPO?=$(AWS_ACCOUNT_ID).dkr.ecr.$(AWS_REGION).amazonaws.com
IMAGE_NAME?=$(COMPONENT)
IMAGE_TAG?=$(GIT_TAG)-eks-${RELEASE_BRANCH}-${RELEASE}
IMAGE?=$(IMAGE_REPO)/$(IMAGE_NAME):$(IMAGE_TAG)

ifeq ($(DEVELOPMENT),true)
	BASE_IMAGE=amazonlinux:2
endif

.PHONY: binaries
binaries:
	build/create_binaries.sh $(REPO) $(CLONE_URL) $(GIT_TAG)

.PHONY: local-images
local-images: binaries
	buildctl \
		build \
		--frontend dockerfile.v0 \
		--opt platform=linux/amd64 \
		--opt build-arg:BASE_IMAGE=$(BASE_IMAGE) \
		--local dockerfile=./docker/linux \
		--local context=. \
		--output type=oci,oci-mediatypes=true,name=$(IMAGE),dest=/tmp/etcd.tar

.PHONY: images
images: binaries
	buildctl \
		build \
		--frontend dockerfile.v0 \
		--opt platform=linux/amd64,linux/arm64 \
		--opt build-arg:BASE_IMAGE=$(BASE_IMAGE) \
		--local dockerfile=./docker/linux \
		--local context=. \
		--output type=image,oci-mediatypes=true,name=$(IMAGE),push=true

.PHONY: docker
docker: binaries
	docker build \
		-t $(IMAGE) \
		--build-arg BASE_IMAGE=$(BASE_IMAGE) \
		--build-arg TARGETARCH=amd64 \
		--build-arg TARGETOS=linux \
		-f ./docker/linux/Dockerfile .

.PHONY: docker-push
docker-push:
	docker push $(IMAGE)

.PHONY: tarballs
tarballs:
	build/create_tarballs.sh $(REPO) $(GIT_TAG)

.PHONY: build
build: local-images tarballs

.PHONY: release
release: images tarballs
	$(BASE_DIRECTORY)/release/copy_artifacts.sh $(COMPONENT) $(RELEASE_BRANCH) $(RELEASE)
	$(BASE_DIRECTORY)/release/s3_sync.sh $(RELEASE_BRANCH) $(RELEASE) $(ARTIFACT_BUCKET) $(REPO)
	echo "Done $(COMPONENT)"

.PHONY: all
all: release

.PHONY: clean
clean:
	rm -rf $(REPO)
	rm -rf "_output"

