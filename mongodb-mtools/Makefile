SHELL := /bin/bash -o pipefail
# Makes a new release for Vendor application when new manifests are added. 

# release_channel: The channel and branch where new versions are released. On a new release to this channel, we increment semver
# app_slug: The current application slug, as identified by the current subdirectory
release_channel := "Stable"
app_slug := "$(shell basename $(CURDIR))"

# channel: The current channel, as identified by the current branch. If provided by environment as GITHUB_BRANCH_NAME, we will use that.  Note the first character is capitalized (e.g., stable->"Stable")
branch_env := ${GITHUB_BRANCH_NAME}
ifdef branch_env
channel := "$(shell ch=$(branch_env);echo $$(tr '[:lower:]' '[:upper:]' <<< $${ch:0:1})$${ch:1})"
else 
channel := "$(shell ch=$$(git rev-parse --abbrev-ref HEAD);echo $$(tr '[:lower:]' '[:upper:]' <<< $${ch:0:1})$${ch:1})"
endif

# When the release is on the primary release channel/branch (e.g., "Stable"), we get the latest version from the latest tag in the repo and increment the semver version (e.g., "1.0.0" -> "1.0.1, or "1.0" -> "1.1")
#    We also push the new latest tag version to git. 
# If the release is not on the primary release branch, our version is based on the latest tag but using the git revision (e.g., something like "1.0.1-Unstable-1967267")
# 
# With github actions, we have the option of explicitly passing PREVIOUS_TAG as an environment variable. If passed, it will be used for current version.  
prev_tag_env := ${PREVIOUS_TAG}
ifdef prev_tag_env
current_version := ${PREVIOUS_TAG}
else 
current_version := "$(shell git describe --tags `git rev-list --tags --max-count=1`)"
endif

ifeq ($(channel), $(release_channel))
version:= "$(shell echo $$(echo $(current_version)) | awk -F. -v OFS=. 'NF==1{print ++$$NF}; NF>1{if(length($$NF+1)>length($$NF))$$(NF-1)++; $$NF=sprintf("%0*d", length($$NF), ($$NF+1)%(10^length($$NF))); print}')"
else
version:=$(current_version)-$(channel)-$(shell git rev-parse HEAD | head -c7)$(shell git diff --no-ext-diff --quiet --exit-code || echo "-dirty")
endif
release_notes := "CLI release of $(version) by ${USER} on $(shell date)"

.PHONE: yarg
yarg: 
	echo $(channel)
	echo $(current_version)
	echo $(version)

.PHONY: deps
deps:
	@if [[ -x ../deps/replicated ]]; then exit 0; else \
	echo '-> Downloading Replicated CLI... '; \
	cd ..; \
	mkdir -p deps/; \
	if [[ "`uname`" == "Linux" ]]; then curl -fsSL https://github.com/replicatedhq/replicated/releases/download/v0.19.0/replicated_0.19.0_linux_amd64.tar.gz | tar xvz -C deps; exit 0; fi; \
	if [[ "`uname`" == "Darwin" ]]; then curl -fsSL https://github.com/replicatedhq/replicated/releases/download/v0.19.0/replicated_0.19.0_darwin_amd64.tar.gz | tar xvz -C deps; exit 0; fi; fi;

.PHONY: lint
lint: check-api-token check-app deps
	../deps/replicated release lint --app $(app_slug) --yaml-dir manifests

.PHONY: check-api-token
check-api-token:
	@if [ -z "${REPLICATED_API_TOKEN}" ]; then echo "Missing REPLICATED_API_TOKEN"; exit 1; fi

.PHONY: check-app
check-app:
	@if [ -z "$(app_slug)" ]; then echo "Missing REPLICATED_APP"; exit 1; fi

.PHONY: list-releases
list-releases: check-api-token check-app deps
	../deps/replicated release ls --app $(app_slug)

.PHONY: release
release: check-api-token check-app deps lint
	../deps/replicated release create \
		--app $(app_slug) \
		--yaml-dir manifests \
		--promote $(channel) \
		--version $(version) \
		--release-notes $(release_notes) \
		--ensure-channel
# Push a new tag if this is a new version. 
ifeq ($(channel), $(release_channel))
	git tag $(version)
	git push origin $(version)
endif

# Use the current branch name for the channel name, 
# and use the git SHA for the release version, 
# adding a "-dirty" suffix to the version label if there are uncomitted changes
gitsha-release:
	@$(MAKE) release

