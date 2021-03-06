SUITE?=metricbeat
TAGS?=
DEVELOPER_MODE?=false
ELASTIC_APM_ACTIVE?=false
FORMAT?=pretty
LOG_INCLUDE_TIMESTAMP?=TRUE
LOG_LEVEL?=INFO
TIMEOUT_FACTOR?=3
#true by default, allowing developers to set SKIP_SCENARIOS=false
SKIP_SCENARIOS?=true
STACK_VERSION?=
PICKLES_VERSION?="2.20.1"
VERSION_VALUE=`cat ../cli/VERSION.txt`
ELASTIC_APM_ENVIRONMENT?=local

ifeq ($(ELASTIC_APM_ACTIVE),true)
ifeq ($(ELASTIC_APM_ENVIRONMENT),ci)
JENKINS_STATS_SECRET?=secret/observability-team/ci/jenkins-stats
export APM_SECRET_TOKEN?=$(shell vault read -field apmServerToken "$(JENKINS_STATS_SECRET)")
export APM_SERVER_URL?=$(shell vault read -field apmServerUrl "$(JENKINS_STATS_SECRET)")
endif

ifeq ($(ELASTIC_APM_ENVIRONMENT),local)
export APM_SECRET_TOKEN?=
export APM_SERVER_URL?=http://localhost:8200
export ELASTIC_APM_GLOBAL_LABELS?=
endif
endif

ifneq ($(TAGS),)
TAGS_FLAG=--tags
ifeq ($(SKIP_SCENARIOS),true)
## We always want to skip scenarios tagged with @skip
TAGS+= && ~skip
endif

# Double quote only if the tags are set
TAGS_VALUE="$(TAGS)"
else
ifeq ($(SKIP_SCENARIOS),true)
TAGS_FLAG=--tags
TAGS_VALUE="~skip"
endif
endif

GO_IMAGE_TAG?='stretch'
GOOS?='linux'
GOARCH?='amd64'

TEST_TIMEOUT?=5m

.PHONT: build-docs
build-docs:
	rm -fr docs
	@docker run --rm --user $$(id -u):$$(id -g) -v $(PWD):/suites docker.elastic.co/observability-ci/picklesdoc:$(PICKLES_VERSION) -f /suites -o /suites/docs --sn "E2E Testing" --sv $(VERSION_VALUE)
	# because pickledocs is a Windows tool, there is a wrong slash.
	mv docs/.\\/index.html docs/index.html
	rm -fr docs/.\\

.PHONY: install
install:
	go get -v -t ./...

.PHONY: install-godog
install-godog: export GO111MODULE := on
install-godog:
	go get -v github.com/cucumber/godog/cmd/godog@v0.11.0

.PHONY: functional-test
functional-test: install-godog
	cd _suites/${SUITE} && \
	OP_LOG_LEVEL=${LOG_LEVEL} \
	OP_LOG_INCLUDE_TIMESTAMP=${LOG_INCLUDE_TIMESTAMP} \
	TIMEOUT_FACTOR=${TIMEOUT_FACTOR} \
	STACK_VERSION=${STACK_VERSION} \
	DEVELOPER_MODE=${DEVELOPER_MODE} \
	ELASTIC_APM_SERVICE_NAME="E2E Tests" \
	ELASTIC_APM_CENTRAL_CONFIG="false" \
	ELASTIC_APM_GLOBAL_LABELS=${ELASTIC_APM_GLOBAL_LABELS} \
	ELASTIC_APM_SERVICE_VERSION="${STACK_VERSION}" \
	ELASTIC_APM_ENVIRONMENT="${ELASTIC_APM_ENVIRONMENT}" \
	ELASTIC_APM_SECRET_TOKEN="${APM_SECRET_TOKEN}" \
	ELASTIC_APM_SERVER_URL="${APM_SERVER_URL}" \
	godog --format=${FORMAT} ${TAGS_FLAG} ${TAGS_VALUE}

.PHONY: lint
lint:
	@docker run -t --rm -v $(PWD):/src -w /src gherkin/lint **/*.feature --disable AvoidOutlineForSingleExample,TooClumsy,TooManySteps,TooManyDifferentTags,TooLongStep

.PHONY: notice
notice:
	@echo "Generating NOTICE"
	go mod tidy
	go mod download
	go list -m -json all | go run go.elastic.co/go-licence-detector \
		-includeIndirect \
		-rules ../notice/rules.json \
		-overrides ../notice/overrides.json \
		-noticeTemplate ../notice/NOTICE.txt.tmpl \
		-noticeOut NOTICE.txt \
		-depsOut ""

.PHONY: unit-test
unit-test:
	gotestsum --format testname -- -count=1 -timeout=$(TEST_TIMEOUT) ./...
	cd _suites && gotestsum --format testname -- -count=1 -timeout=$(TEST_TIMEOUT) ./...

## Test examples

.PHONY: fleet-fleet
fleet-fleet:
	SUITE="fleet" TAGS="fleet_mode_agent" TIMEOUT_FACTOR=3 LOG_LEVEL=TRACE DEVELOPER_MODE=true $(MAKE) functional-test

.PHONY: fleet-fleet-ci-snapshots
fleet-fleet-ci-snapshots:
	SUITE="fleet" TAGS="fleet_mode_agent" TIMEOUT_FACTOR=3 LOG_LEVEL=TRACE BEATS_USE_CI_SNAPSHOTS=true DEVELOPER_MODE=true GITHUB_CHECK_SHA1=a1962c8864016010adcde9f35bd8378debb4fbf7 $(MAKE) functional-test

.PHONY: fleet-fleet-pr-ci-snapshots
fleet-fleet-pr-ci-snapshots:
	SUITE="fleet" TAGS="fleet_mode_agent" TIMEOUT_FACTOR=3 LOG_LEVEL=TRACE BEATS_USE_CI_SNAPSHOTS=true DEVELOPER_MODE=true BEAT_VERSION=pr-14954 $(MAKE) functional-test

.PHONY: fleet-nightly
fleet-nightly:
	SUITE="fleet" TAGS="fleet_mode_agent && nightly" TIMEOUT_FACTOR=3 LOG_LEVEL=TRACE DEVELOPER_MODE=true $(MAKE) functional-test

.PHONY: fleet-nightly-ci-snapshots
fleet-nightly-ci-snapshots:
	SUITE="fleet" TAGS="fleet_mode_agent && nightly" TIMEOUT_FACTOR=3 LOG_LEVEL=TRACE BEATS_USE_CI_SNAPSHOTS=true DEVELOPER_MODE=true GITHUB_CHECK_SHA1=a1962c8864016010adcde9f35bd8378debb4fbf7 $(MAKE) functional-test
