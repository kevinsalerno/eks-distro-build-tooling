AWS_ACCOUNT_ID?=$(shell aws sts get-caller-identity --query Account --output text)
AWS_REGION?=us-west-2

IMAGE_REPO?=$(AWS_ACCOUNT_ID).dkr.ecr.$(AWS_REGION).amazonaws.com
IMAGE_NAME?=builder-base
# This tag is overwritten in the prow job to point to the PR branch commit hash (presubmit)
# or the base branch commit hash (postsubmit)
IMAGE_TAG?=latest

AL_TAG?=2
LATEST?=latest

ifeq (2022,$(AL_TAG))
	LATEST:=$(LATEST).2022
else
	# tag al2 build with latest.2 and latest for backward compat
	LATEST:=$(LATEST) $(LATEST).2
endif

IMAGE?=$(IMAGE_REPO)/$(IMAGE_NAME):$(IMAGE_TAG)

LATEST_TAGS=$(foreach tag,$(LATEST),$(IMAGE_REPO)/$(IMAGE_NAME):$(tag))
LATEST_IMAGE=$(shell echo $(LATEST_TAGS) | sed "s/ \+/,/g")

BASE_IMAGE_REPO?=public.ecr.aws/eks-distro-build-tooling
BASE_IMAGE_NAME?=eks-distro-minimal-base-kind
BASE_IMAGE?=$(BASE_IMAGE_REPO)/$(BASE_IMAGE_NAME):$(call BASE_TAG_FROM_TAG_FILE,$(BASE_IMAGE_NAME))

BUILDKIT_OUTPUT=type=image,oci-mediatypes=true,\"name=$(IMAGE),$(LATEST_IMAGE)\",push=true
BUILDKIT_PLATFORMS?=linux/amd64,linux/arm64

MAKE_ROOT=$(shell cd "$(shell dirname "${BASH_SOURCE[0]}")" && pwd -P)

define BASE_TAG_FROM_TAG_FILE
$(shell yq e ".al$(AL_TAG).$(1)" $(MAKE_ROOT)/../EKS_DISTRO_TAG_FILE.yaml)
endef

.PHONY: buildkit-check
buildkit-check:
	$(MAKE_ROOT)/../scripts/buildkit_check.sh

.PHONY: copy-generate-attribution
copy-generate-attribution:
	cp -rf ../generate-attribution ./

.PHONY: remove-generate-attribution
remove-generate-attribution:
	rm -rf ./generate-attribution

.PHONY: local-images
local-images: BUILDKIT_OUTPUT=type=tar,dest=/dev/null
local-images: BUILDKIT_PLATFORMS=linux/amd64
local-images: images
	
.PHONY: images
images: buildkit-check
	buildctl \
		build \
		--frontend dockerfile.v0 \
		--opt platform=$(BUILDKIT_PLATFORMS) \
		--opt build-arg:BASE_IMAGE=$(BASE_IMAGE) \
		--opt build-arg:GOPROXY=$(GOPROXY) \
		--local dockerfile=./ \
		--local context=. \
		--progress plain \
		--output $(BUILDKIT_OUTPUT)
	if [ "$(AL_TAG)" = "2" ]; then \
		./update_base_image.sh $(IMAGE_TAG); \
	fi
	

# for local development only
docker: copy-generate-attribution
	docker build \
		-t $(IMAGE) \
		--build-arg BASE_IMAGE=$(BASE_IMAGE) \
		--build-arg TARGETARCH=amd64 \
		--build-arg TARGETOS=linux \
		-f Dockerfile .
	$(MAKE) remove-generate-attribution

.PHONY: build
build: copy-generate-attribution local-images remove-generate-attribution

.PHONY: release
release: copy-generate-attribution images remove-generate-attribution

.PHONY: all
all: release
